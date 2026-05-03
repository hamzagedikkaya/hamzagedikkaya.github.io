---
layout: post
title: "EagerEye: Catching the N+1s Bullet Misses — with Static Analysis"
author: Hamza Gedikkaya
categories: 
  - Rails Performance
  - Static Analysis
  - Tools
excerpt_image: /assets/images/posts/third_gif.gif
banner:
  image: /assets/images/posts/third_post.jpg
  height: "50vh"
tags: 
  - Ruby on Rails
  - Performance
  - ActiveRecord
  - Static Analysis
  - Open Source
---

A few months ago I wrote [Beyond N+1: Hidden Performance Traps and Fixes]({% post_url 2025-12-06-beyond_n+1 %}) about the performance killers Bullet can't catch — queries inside custom methods, serializer-induced query explosions, callback-driven inserts, and so on. The post ended with a list of techniques and tools to work around Bullet's blind spots, but it didn't really answer the obvious follow-up question: *can we catch these automatically, in CI, without running a single test?*

That's the gap [EagerEye](https://github.com/hamzagedikkaya/eager_eye) tries to close. It's a static analyzer for Rails — no DB, no Rails boot, no runtime hooks. It parses your Ruby files into an AST and looks for patterns that turn into N+1 queries at runtime. In this post I'll walk through the design decisions behind it, show what it catches that Bullet can't, share the real-world numbers from running it on production codebases, and explain how to wire it into CI in five lines of YAML.

---

## Table of Contents

1. [Why Static Analysis?](#1-why-static-analysis)
2. [What EagerEye Catches](#2-what-eagereye-catches)
3. [Design Decisions: Why This and Not That](#3-design-decisions-why-this-and-not-that)
4. [Real-World Results: Two Production Codebases](#4-real-world-results-two-production-codebases)
5. [Installation and First Scan](#5-installation-and-first-scan)
6. [CI Integration: Five Lines of YAML](#6-ci-integration-five-lines-of-yaml)
7. [VS Code Extension: Same Engine, in Your Editor](#7-vs-code-extension-same-engine-in-your-editor)
8. [Suppressing False Positives](#8-suppressing-false-positives)
9. [Known Limitations](#9-known-limitations)
10. [Roadmap and How to Contribute](#10-roadmap-and-how-to-contribute)

---

## 1. Why Static Analysis?

Bullet is a runtime tool. It hooks into ActiveRecord's association loading mechanism and notices when you call an association that wasn't preloaded. This is brilliant when it works, but it has a fundamental constraint: **the code path has to actually run**.

Most teams I've worked with have decent test coverage for their happy paths but limited coverage for everything else: admin actions, error branches, rarely-hit feature flags, background jobs that only fire on specific events. Bullet sees nothing in any of those code paths. Worse, if the test happens to use fewer records than would trigger an N+1 in production (`create(:user, :with_posts)` in a fixture vs. 10,000 users in prod), you can pass tests with hidden N+1s.

Static analysis solves a different problem. It reads every line of code, regardless of whether it ever executes. It finds patterns. It doesn't need fixtures, doesn't need a DB, doesn't need Redis. It runs in milliseconds per file, not minutes.

The trade-off is that it can't verify anything. A heuristic that matches `posts.each { |p| p.author }` might be flagging a real N+1 — or it might be flagging code where `posts` was already preloaded somewhere static analysis can't see. The only honest path forward is to design the tool around minimizing false positives, even if it costs some false negatives.

That principle — **a warning you can trust is more useful than a warning you have to investigate** — is what shaped the entire design.

---

## 2. What EagerEye Catches

EagerEye ships 11 detectors. Some overlap with Bullet (the simple loop case), most don't.

### LoopAssociation — the obvious one

```ruby
posts.each do |post|
  post.author.name      # query per post
  post.comments.count   # another query per post
end
```

EagerEye flags both lines and suggests `.includes(:author, :comments)`. So does Bullet, *if* this code is exercised by a test that loops over enough posts. But if the loop is in an admin export controller that nobody tests, Bullet stays silent. EagerEye doesn't care — it sees the loop and the association access regardless.

### CustomMethodQuery — the one Bullet can't see

```ruby
class User < ApplicationRecord
  has_many :teams

  def supports?(team_name)
    teams.where(name: team_name).exists?
  end
end

@users.each { |user| user.supports?("Lakers") }
```

This is the example I opened the [previous post]({% post_url 2025-12-06-beyond_n+1 %}#2-hidden-n1s-in-custom-methods) with. Bullet can't catch it because `teams.where(...)` bypasses the association loading hook. EagerEye scans every model file once, builds a map of which methods contain query calls, then flags any iteration that calls one of those methods on the iteration variable.

### SerializerNesting — query explosions in JSON

```ruby
class PostSerializer < Blueprinter::Base
  field :author_name { |post| post.author.name }
end

# Controller — no preload
render json: PostSerializer.render(@posts)
```

Serializers are an N+1 graveyard. They run once per record, often access nested associations, and the controller doesn't always know which fields the serializer touches. EagerEye scans Blueprinter, ActiveModel::Serializer, and Alba blocks for nested association access and suggests preloading at the source.

### CountInIteration — the `.count` vs `.size` trap

```ruby
@users = User.includes(:posts)
@users.each { |user| user.posts.count }   # SELECT COUNT(*) per user, even though posts are loaded
```

`.count` always queries. `.size` uses the loaded array if available. Bullet doesn't catch this because the association *is* preloaded — you're just not using the preload. EagerEye flags every `.count` on an association inside an iteration block.

### CallbackQuery — the silent killer

```ruby
class Order < ApplicationRecord
  after_create :notify_subscribers

  def notify_subscribers
    customer.followers.each do |follower|
      follower.notifications.create!(...)   # N inserts + N queries per save
    end
  end
end
```

`Order.import(big_array)` triggers the callback for every record. Per record, you get an iteration that fires queries. Bullet usually doesn't run during background jobs and doesn't track `create!` patterns. EagerEye specifically scans `after_*` / `before_*` / `around_*` callback bodies for iteration-driven queries.

### And six more

`MissingCounterCache`, `PluckToArray`, `DelegationNPlusOne`, `DecoratorNPlusOne`, `ScopeChainNPlusOne`, `ValidationNPlusOne` — each targets a pattern from the previous post. The full list with code samples lives in the [README](https://github.com/hamzagedikkaya/eager_eye#what-it-detects).

---

## 3. Design Decisions: Why This and Not That

### Why AST instead of regex

The naive approach to "find loops with association calls" is a regex like `/each.*\.(\w+)\.\w+/`. This breaks on the first multi-line block, misses Hash literals, and confuses string interpolation with method calls. AST parsing means we work with the actual structure Ruby sees: `:block` nodes contain a `:send` (the iteration call), an `:args` list, and a body of statements. Walking that tree is annoying but reliable.

The implementation uses [`whitequark/parser`](https://github.com/whitequark/parser), the same parser RuboCop uses. Ruby 3.1+ syntax is supported.

### Why per-method scope tracking

One of the gnarlier bugs in early versions was this:

```ruby
def index
  invoices = Invoice.includes(:customer, :merchant).where(active: true)
  @data = invoices.map { |i| [i.customer.name, i.merchant.name] }
end

def some_other_action
  invoices = Invoice.where(id: params[:ids])  # no includes here
  invoices.update_all(status: 'archived')
end
```

The first version of EagerEye tracked `invoices` globally across the file. So when it processed `def some_other_action`, the `invoices` variable assignment overwrote the preload information from `def index`, and the iteration in `index` started getting flagged for N+1 even though `:customer` and `:merchant` were preloaded.

The fix was treating each `:def` body as an independent scope: variables inherit a snapshot from the enclosing scope, but writes inside a method don't leak out. This is closer to how Ruby actually works and eliminated about 19 false positives on the iwallet codebase alone.

### Why caller-method preload tracking

A common Rails pattern is to extract serialization into a helper:

```ruby
def index
  @users = User.includes(:profile, :organization)
  @data = prepare_data(@users)
end

private

def prepare_data(users)
  users.map { |u| [u.profile.bio, u.organization.name] }
end
```

The iteration is in `prepare_data`. Static analysis can see that `users` is a method parameter — but without inter-procedural analysis, it can't know that the only caller (`index`) preloads `:profile` and `:organization`.

EagerEye does a two-pass analysis per class: pass 1 collects every self-call between sibling methods along with the caller's variable state at the call site; pass 2 processes each method with its parameters seeded from the merged caller context. If at least one caller preloads `:profile`, the parameter inherits that preload. This handles the helper-method pattern that's everywhere in Rails controllers.

### Why prefer false negatives over false positives

This is the philosophical decision the entire tool rests on. Every heuristic has knobs. You can lean toward "flag anything that looks suspicious" — high recall, lots of noise, users start ignoring warnings within a week. Or you can lean toward "only flag what we're confident about" — lower recall, but every warning is actionable.

EagerEye picks the second path. When in doubt, suppress. When a method has multiple callers and only some preload an association, treat it as preloaded (better to miss a real N+1 than to flag a non-issue). When a model isn't in the parsed set, defer to a small hardcoded list of well-known association names rather than flagging every method call on a loop variable.

The result: in the two production codebases I tested it on, the false positive rate is under 1%. Every flag is worth investigating.

---

## 4. Real-World Results: Two Production Codebases

I ran EagerEye through two production Rails apps I work on. Both are 5+ years old, multi-thousand-file codebases with mature test suites. Bullet runs in their dev environment and catches the N+1s that show up in tests. Here's what EagerEye found on top of that.

### Codebase A (~160 files affected)

| Detector | Issues |
|---|---:|
| LoopAssociation | 404 |
| CustomMethodQuery | 184 |
| SerializerNesting | 150 |
| CallbackQuery | 33 |
| PluckToArray | 24 |
| ValidationNPlusOne | 17 |
| MissingCounterCache | 8 |
| ScopeChainNPlusOne | 4 |
| CountInIteration | 2 |
| DelegationNPlusOne | 1 |
| **Total** | **827** |

I sampled ~50 issues across detectors and manually verified them against the actual code paths. Real positives: 49. False positives: 1 (and that one was in a code path Bullet also can't see — it's a "controller passes preloaded relation to a service object in another file" case).

### Codebase B (~70 files affected)

| Detector | Issues |
|---|---:|
| SerializerNesting | 118 |
| LoopAssociation | 75 |
| CustomMethodQuery | 17 |
| PluckToArray | 5 |
| CallbackQuery | 4 |
| ScopeChainNPlusOne | 1 |
| **Total** | **220** |

Same sampling exercise: 100% real positives in my sample.

### What this means in practice

These aren't all "production-blocking" issues. Some are admin-export controllers that run once a week. Some are background jobs that happen to be fast despite the N+1 because each query is tiny. But every single one is a place where someone made a decision (intentionally or not) to leave a query-per-iteration pattern in the code, and they probably didn't know.

The really useful warnings are the ones in hot paths. SerializerNesting in `Api::V2::ProductsSerializer` rendered millions of times a day is a different problem than a one-off `db:seed` script. EagerEye flags both, and you decide which to fix.

---

## 5. Installation and First Scan

```ruby
# Gemfile
gem "eager_eye", group: :development
```

```bash
bundle install
```

That's it. No Rails initializer, no config file. From your project root:

```bash
eager_eye          # scans app/ by default
eager_eye app/controllers app/serializers   # specific paths
eager_eye --format json                     # for CI tools to parse
eager_eye --only loop_association,serializer_nesting   # specific detectors
```

A fresh scan of a typical Rails app finishes in 2-5 seconds.

If you want to suppress detectors or set per-detector severity, generate a config file:

```bash
rails g eager_eye:install
```

That creates `.eager_eye.yml`:

```yaml
excluded_paths:
  - app/legacy/**
  - lib/tasks/**

severity_levels:
  loop_association: error
  missing_counter_cache: info

min_severity: warning
fail_on_issues: true
```

`fail_on_issues: true` makes the CLI exit with a non-zero status when issues are found — the foundation for CI integration.

---

## 6. CI Integration: Five Lines of YAML

The whole point of static analysis is that it runs without infrastructure. Here's a complete GitHub Actions workflow:

```yaml
name: EagerEye
on: [pull_request]

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.3"
      - run: gem install eager_eye
      - run: eager_eye app/
```

No DB setup, no `bundle install` of your full Gemfile, no fixtures. The whole job runs in under a minute. If new code introduces a flagged pattern, the build fails and the PR is blocked.

For teams that want non-blocking warnings instead, swap the last line for:

```yaml
- run: eager_eye app/ --format json > report.json
- run: |
    issues=$(ruby -rjson -e 'puts JSON.parse(File.read("report.json"))["summary"]["total_issues"]')
    [ "$issues" -gt 0 ] && echo "::warning::Found $issues potential N+1 issues" || true
```

This uses GitHub Actions' `::warning::` annotation syntax, which surfaces the issue count directly on the PR without failing the build. Useful during a gradual adoption phase where you want visibility but not enforcement.

---

## 7. VS Code Extension: Same Engine, in Your Editor

For the development loop, a CLI run after every change is friction. EagerEye also ships as a [VS Code extension](https://marketplace.visualstudio.com/items?itemName=hamzagedikkaya.eager-eye) that runs on save and surfaces issues inline:

- Squiggly underline on the offending line
- Hover for the explanation and suggestion
- Quick Fix actions for common patterns (`.pluck(:id)` → `.select(:id)`, etc.)
- Status bar showing total issue count for the current file

The extension is a thin wrapper around the gem — it shells out to the `eager_eye` binary on save and parses the JSON output. Same analysis engine, same detection, just a smoother feedback loop.

Recommended workflow: extension during development for fast iteration, CLI in CI to gate PRs.

---

## 8. Suppressing False Positives

When EagerEye gets it wrong (rare, but it happens), you suppress like RuboCop:

```ruby
user.posts.count  # eager_eye:disable CountInIteration

# eager_eye:disable-next-line LoopAssociation
@users.each { |u| u.profile }

# eager_eye:disable LoopAssociation, SerializerNesting
@users.each { |u| u.posts.each { |p| p.author } }
# eager_eye:enable LoopAssociation, SerializerNesting

# Whole file (must be in first 5 lines)
# eager_eye:disable-file CustomMethodQuery

# With explanation
user.posts.count  # eager_eye:disable CountInIteration -- using counter_cache
```

The `-- reason` syntax is borrowed from RuboCop and is purely documentation — the linter doesn't enforce it but reviewers will appreciate it.

---

## 9. Known Limitations

Static analysis isn't magic. Three things EagerEye can't do today:

**Cross-file flow tracking.** EagerEye propagates preload context across method calls within the same class. If a controller calls a service object in a different file (`OrderProcessor.new(orders).call`), the analyzer can't see that the orders were preloaded. Same applies to renderable view partials.

**Runtime metadata.** EagerEye doesn't read your DB schema, doesn't know if a column has an index, doesn't know how many records actually live in production. A `Post.where(active: true).each` looks identical whether `active` is on 10 records or 10 million. Bullet plus production monitoring (Skylight, Scout, NewRelic) cover this.

**Heuristic association detection.** When a method is called on an iteration variable but the variable's model class can't be inferred, EagerEye falls back to a small hardcoded list of common association names (`author`, `user`, `posts`, etc.). This can miss exotic naming, and very rarely it can over-flag (a column happens to share a name with a common association). The hardcoded list errs on the side of suppression.

The honest summary: use EagerEye alongside Bullet, not instead of it. Static catches code paths Bullet can't reach; runtime catches what static can't see. They're complementary.

---

## 10. Roadmap and How to Contribute

The thing I most want to add next is **inter-file call graph tracking** — propagating preload context not just across same-class methods but across `include`d modules and called service objects. The current implementation handles intra-class flow well; cross-file is the main remaining false-positive source.

Beyond that:

- A `--baseline` mode that snapshots existing issues and only fails CI on *new* ones (so you can adopt EagerEye on a brownfield project without fixing 800 existing warnings on day one)
- Integration with [Reek](https://github.com/troessner/reek) and [RuboCop](https://rubocop.org/) for unified linter output
- A web dashboard for tracking issue trends over time

If any of these sound useful, the gem is MIT-licensed and open to PRs:

- Repo: [github.com/hamzagedikkaya/eager_eye](https://github.com/hamzagedikkaya/eager_eye)
- VS Code extension: [github.com/hamzagedikkaya/eager_eye_vscode](https://github.com/hamzagedikkaya/eager_eye_vscode)
- Issues and feature requests welcome.

---

## Conclusion

In the [previous post]({% post_url 2025-12-06-beyond_n+1 %}) I argued that Bullet only catches the tip of the N+1 iceberg. EagerEye is my attempt at catching most of the rest, automatically, in CI, on every PR — without DB infrastructure, without test fixtures, and without learning a new query DSL.

It won't catch everything. Static analysis fundamentally can't. But it shifts the detection point from "after deployment, when production starts paging" to "before merge, when the developer can still fix it cheaply." That shift is most of what makes a tool useful.

If you try it on a real codebase and find it useful — or find a false positive — open an issue. The whole reason I built this is that the existing tooling left a gap, and the only way to close that gap is to keep iterating on real codebases.

---

## Resources

- [EagerEye on RubyGems](https://rubygems.org/gems/eager_eye)
- [EagerEye on GitHub](https://github.com/hamzagedikkaya/eager_eye)
- [VS Code Extension](https://marketplace.visualstudio.com/items?itemName=hamzagedikkaya.eager-eye)
- [Beyond N+1: Hidden Performance Traps and Fixes]({% post_url 2025-12-06-beyond_n+1 %}) (the post that motivated this tool)
- [Bullet Gem](https://github.com/flyerhzm/bullet)
- [Prosopite](https://github.com/charkost/prosopite)
- [Parser Gem](https://github.com/whitequark/parser) (the AST library EagerEye is built on)
- [RuboCop](https://github.com/rubocop/rubocop) (architectural inspiration for the suppression syntax)
