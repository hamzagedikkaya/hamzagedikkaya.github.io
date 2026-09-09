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

A few months ago I wrote [Beyond N+1: Hidden Performance Traps and Fixes]({% post_url 2025-12-06-beyond_n+1 %}) about the performance killers Bullet can't catch — queries inside custom methods, serializer-induced query explosions, callback-driven inserts, `.count` on a collection you already preloaded. The post ended with a list of workarounds, but it dodged the obvious follow-up: *can we catch these automatically, in CI, without running a single test?*

[EagerEye](https://github.com/hamzagedikkaya/eager_eye) is my answer. It's a static analyzer for Rails: no database, no Rails boot, no runtime hooks. It parses your Ruby into an AST and looks for the patterns that become N+1 queries at runtime. This post covers the design decisions behind it, what it catches that Bullet can't, the honest numbers from running it on two production codebases — including the false-positive rate I got wrong the first time — and how to wire it into CI in a handful of lines.

> *Updated September 2026 for EagerEye 1.3.3.* Since the original version of this post: baseline mode shipped (it was on the roadmap), the analyzer became schema-aware, serializer detection learned to look at render sites, and a large hand-verification pass cut false positives by roughly half. Sections 3, 4, 6 and 10 changed the most.

---

## Table of Contents

1. [Why Static Analysis?](#1-why-static-analysis)
2. [What EagerEye Catches](#2-what-eagereye-catches)
3. [Design Decisions: Why This and Not That](#3-design-decisions-why-this-and-not-that)
4. [Real-World Results: Two Production Codebases](#4-real-world-results-two-production-codebases)
5. [Installation and First Scan](#5-installation-and-first-scan)
6. [CI Integration and Baseline Mode](#6-ci-integration-and-baseline-mode)
7. [RSpec Matcher and Auto-fix](#7-rspec-matcher-and-auto-fix)
8. [VS Code Extension: Same Engine, in Your Editor](#8-vs-code-extension-same-engine-in-your-editor)
9. [Suppressing False Positives](#9-suppressing-false-positives)
10. [Known Limitations](#10-known-limitations)
11. [Roadmap and How to Contribute](#11-roadmap-and-how-to-contribute)

---

## 1. Why Static Analysis?

Bullet is a runtime tool. It hooks into ActiveRecord's association loading and notices when you touch an association that wasn't preloaded. Brilliant when it fires — but it has one hard constraint: **the code path has to actually run.**

Most teams have good coverage of their happy paths and thin coverage of everything else: admin actions, error branches, feature flags, jobs that only fire on specific events. Bullet sees nothing in any of those. Worse, a test that creates two records will never hit the threshold that ten thousand production rows would, so you can pass a green suite with a hidden N+1 in it.

Static analysis reads every line whether or not it ever executes. It needs no fixtures, no database, no Redis, and it runs in seconds on a whole codebase rather than minutes per test run.

The trade-off is that it can't *verify* anything. A pattern that matches `posts.each { |p| p.author }` may be a real N+1 — or `posts` may have been preloaded somewhere the analyzer can't see. So the only honest way to build the tool is to design it around minimizing false positives, even at the cost of some false negatives:

> **A warning you can trust is worth more than a warning you have to investigate.**

That sentence shaped every decision below.

---

## 2. What EagerEye Catches

EagerEye ships **11 detectors**. One overlaps with Bullet; the rest target patterns Bullet is structurally blind to.

### LoopAssociation — the obvious one

```ruby
posts.each do |post|
  post.author.name      # query per post
  post.comments.count   # another query per post
end
```

Flagged with a suggestion to add `.includes(:author, :comments)`. Bullet catches this too — *if* a test loops over enough posts. If the loop lives in an admin export nobody tests, only EagerEye sees it.

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

This is the example the [previous post]({% post_url 2025-12-06-beyond_n+1 %}#2-hidden-n1s-in-custom-methods) opened with. `teams.where(...)` builds a new relation and bypasses the association cache, so Bullet never hears about it. EagerEye scans every model once, builds a map of which methods contain query calls, then flags any iteration that calls one of those methods on the loop variable — scoped per model, so `obj.foo` isn't flagged just because some *other* model has a `def foo` with a query in it.

### SerializerNesting — query explosions in JSON

```ruby
class PostSerializer < Blueprinter::Base
  field(:author_name) { |post| post.author.name }
end

render json: PostSerializer.render(@posts)   # no preload
```

Serializers run once per record, reach into associations, and the controller rarely knows which fields they touch. EagerEye understands Blueprinter, ActiveModel::Serializers and Alba, and — since 1.3.1 — also looks at *where* a serializer is rendered, so it stays quiet when every render site preloads the association (more in §3).

### CountInIteration — the `.count` vs `.size` trap

```ruby
@users = User.includes(:posts)
@users.each { |user| user.posts.count }   # SELECT COUNT(*) per user, preload ignored
```

`.count` always queries; `.size` uses the loaded array. Bullet can't flag this because the association *is* preloaded — you're just not using the preload.

### CallbackQuery — the silent killer

```ruby
class Order < ApplicationRecord
  after_create :notify_subscribers

  def notify_subscribers
    customer.followers.each { |f| f.notifications.create!(...) }   # N inserts + N queries per save
  end
end
```

`Order.import(big_array)` fires this once per record. Bullet rarely runs during jobs and doesn't track `create!` patterns; EagerEye specifically inspects `before_*` / `after_*` / `around_*` bodies for iteration-driven queries.

### And six more

| Detector | Catches |
|---|---|
| `MissingCounterCache` | `.count` / `.size` on an association inside a loop where a counter cache would remove the query entirely |
| `PluckToArray` | `.pluck(:id)` fed into `where(id: ...)` instead of a subquery; an unscoped `.all.pluck` is escalated to **error** |
| `DelegationNPlusOne` | `delegate :name, to: :user` — reads that look like attributes but load an association per row |
| `DecoratorNPlusOne` | Draper / SimpleDelegator / presenter methods that touch associations, called after `.decorate` with no preload |
| `ScopeChainNPlusOne` | Named scopes (`.recent`, `.active`) on an association inside a loop — a query hidden behind a friendly name |
| `ValidationNPlusOne` | `Model.create` in a loop where the model has a uniqueness validation — a `SELECT` before every `INSERT` |

All detectors understand `each`, `map`, `find_each`, `in_batches`, `each_with_object`, `reduce`, and friends. Ruby files, `.jbuilder` templates, and Ruby 3.1+ syntax are supported. Full examples for every detector are in the [README](https://github.com/hamzagedikkaya/eager_eye#what-it-detects).

---

## 3. Design Decisions: Why This and Not That

### AST, not regex

The naive approach — `/each.*\.(\w+)\.\w+/` — dies on the first multi-line block, misreads string interpolation, and can't tell a hash key from a method call. EagerEye uses [`whitequark/parser`](https://github.com/whitequark/parser), the same AST library RuboCop is built on. A `:block` node has a `:send` (the iteration call), an `:args` list, and a body; walking that tree is tedious but reliable, and it's the only way to reason about *which variable* is being iterated and *where it came from*.

### Per-method scope for variable tracking

One of the nastier early bugs:

```ruby
def index
  invoices = Invoice.includes(:customer, :merchant).where(active: true)
  @data = invoices.map { |i| [i.customer.name, i.merchant.name] }
end

def archive
  invoices = Invoice.where(id: params[:ids])   # no includes
  invoices.update_all(status: "archived")
end
```

The first version tracked `invoices` across the whole file, so the assignment in `archive` overwrote the preload information from `index`, and a perfectly preloaded loop got flagged. The fix: each `:def` body is an independent scope that inherits a snapshot from its parent but whose writes never leak out. That alone removed 19 false positives on one codebase.

### Caller-to-callee preload propagation

Rails controllers extract helpers constantly:

```ruby
def index
  @users = User.includes(:profile, :organization)
  @rows  = prepare_rows(@users)
end

private

def prepare_rows(users)
  users.map { |u| [u.profile.bio, u.organization.name] }
end
```

The loop is in `prepare_rows`, whose parameter has no preload context on its own. EagerEye does two passes per class: the first records every self-call along with the caller's variable state at the call site; the second analyzes each method with its parameters seeded from the merged caller context. If any caller preloads `:profile`, the parameter inherits it. Merging permissively (any preloading caller suppresses the warning) is a deliberate false-negative trade.

### Schema awareness without a database connection (1.3.1)

When the analyzer can't infer the model of a loop variable, it used to fall back to guessing whether `record.vat_rate` is a column or an association. Now it reads `db/schema.rb` — found by walking up from the scanned path — and learns every table's real columns. A method whose name the schema knows as a column is never flagged as a query. It still never connects to a database, parses SQL, or needs Rails loaded; if there's no `schema.rb`, the guard is simply off.

### Render-site awareness for serializers (1.3.1)

`SerializerNesting` used to flag every nested association access in every serializer, regardless of what the controller did. Now a second parser scans render sites — `PostBlueprint.render(...)`, AMS `serializer:` / `each_serializer:` — and records, per serializer and view, which associations are preloaded and whether the serializer only ever receives single records. If every render site preloads `:author`, the warning is suppressed. A view EagerEye never sees rendered is still reported: it never concludes "safe" from a lack of evidence.

### One warning per association per iteration

A memoized `belongs_to` read five times in one loop body used to produce five warnings. Repeated reads hit Rails' instance-level association cache, not the database, so `LoopAssociation` now reports each association once per iteration. Associations used as the base of a query chain (`x.assoc.find_by`, `x.assoc.where`) still report every occurrence, because those *do* re-query.

### Prefer false negatives over false positives

Every heuristic has a knob. Turn it toward "flag anything suspicious" and users learn to ignore the tool within a week. Turn it toward "only flag what we're confident about" and every warning becomes actionable. EagerEye picks the second setting everywhere: when a method has some preloading callers and some not, treat it as preloaded; when a model isn't in the parsed set and the schema doesn't help, defer to a short list of well-known association names; when a save skips validations, don't flag `ValidationNPlusOne`. Section 4 shows what that policy costs and buys.

---

## 4. Real-World Results: Two Production Codebases

I ran EagerEye on two production Rails apps I work on. Both are 5+ years old, multi-thousand-file codebases with mature test suites and Bullet enabled in development. This is what the analyzer found *on top of* what Bullet already reports.

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

### The number I got wrong

When this post was first published, I had sampled about 50 findings by hand and found one false positive. I wrote "under 1%". Then I did the thing I should have done first and hand-checked **all ~1,080 findings** across both codebases.

| | Before 1.3.1 | After 1.3.1 |
|---|---:|---:|
| Hand-verified findings | ~1,080 | ~1,080 |
| False positives | 339 (~31%) | 158 (~15%) |
| Real findings lost by the changes | — | 5 |

Thirty-one percent is not "under one percent". The gap came from exactly the places you'd expect: column reads on receivers whose model couldn't be inferred (`record.comsn_rate`), the same memoized `belongs_to` reported five times, and serializers flagged even when every controller preloaded the association. Those three findings became the schema guard, per-iteration dedup, and render-site awareness in §3, and the pass cut false positives by 53% while losing five real findings out of roughly 740.

Fifteen percent still isn't zero, and the remaining noise is concentrated in one shape: cross-file flow, where a controller preloads a relation and hands it to a service object in another file (§10). The detectors that don't depend on that — `CountInIteration`, `PluckToArray`, `CallbackQuery`, `ValidationNPlusOne` — are close to 100% precise on this dataset.

### What the real findings mean

Not all of them are production emergencies. Some are weekly admin exports; some are jobs where each query is tiny. But every one is a place where a query-per-iteration pattern was left in the code, almost always without anyone knowing. The valuable ones are in hot paths — a `SerializerNesting` hit in an API serializer rendered millions of times a day is a different animal from a `db:seed` script. EagerEye flags both; deciding which to fix is your job.

---

## 5. Installation and First Scan

```ruby
# Gemfile
gem "eager_eye", group: :development
```

```bash
bundle install
eager_eye               # scans app/ by default
```

No initializer, no config file. A typical Rails app scans in 2–5 seconds. Output looks like this:

```text
app/controllers/posts_controller.rb
  Line 15: [LoopAssociation] Potential N+1 query: `post.author` called inside iteration
           Suggestion: Use `includes(:author)` on the collection before iterating

  Line 23: [MissingCounterCache] `.count` called on `comments` may cause N+1 queries
           Suggestion: Add `counter_cache: true` to the belongs_to association

Total: 2 issues (2 warnings, 0 errors)
```

Useful flags:

```bash
eager_eye app/controllers app/serializers          # specific paths
eager_eye --format json                            # machine-readable, for CI
eager_eye --only loop_association,serializer_nesting
eager_eye --exclude "app/legacy/**"
eager_eye --min-severity error                     # ignore warnings and info
eager_eye --no-fail                                # always exit 0
```

One tip that matters more than any flag: **scan a path that contains `models/`** — `app/`, not `app/controllers`. Model metadata (associations, `delegate`, `scope`, uniqueness validations) is collected from `<first path>/models/**`, and three detectors plus preload-aware association tracking depend on it.

### Configuration file (rake tasks)

`rails g eager_eye:install` creates `.eager_eye.yml`, which the `rake eager_eye:analyze` and `rake eager_eye:json` tasks load from `Rails.root`:

```yaml
excluded_paths:
  - app/legacy/**
  - lib/tasks/**

enabled_detectors:       # default: all 11
  - loop_association
  - serializer_nesting
  - custom_method_query

app_path: app
fail_on_issues: true
```

The `eager_eye` CLI intentionally does *not* read this file — it takes the equivalent flags instead, so a CI step is self-describing. Programmatic configuration via `EagerEye.configure { |c| ... }` is also available for projects that wrap the analyzer in their own runner.

---

## 6. CI Integration and Baseline Mode

The point of static analysis is that CI needs no infrastructure. A complete GitHub Actions workflow:

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

No database service, no `bundle install` of your full Gemfile, no fixtures. The job finishes in well under a minute and fails the PR if any issue is found.

### Baseline mode: adopting EagerEye on a brownfield app

The workflow above is unusable on an existing codebase — see the 827 findings in §4. Nobody is going to fix 827 issues before turning on a linter. So EagerEye 1.3.1 added the feature that was on the roadmap when this post first went out:

```bash
# once, locally: snapshot today's issues
eager_eye app/ --format json > .eager_eye_baseline.json
git add .eager_eye_baseline.json

# in CI: fail only on issues that are NOT in the baseline
eager_eye app/ --baseline .eager_eye_baseline.json
```

Existing issues are accepted as debt; the build fails only on **regressions** — a new N+1 introduced by the PR. As you pay down the debt, regenerate the baseline. The baseline is a plain `--format json` report, so there's nothing new to learn; the match key is `(detector, file, line, message, severity, suggestion)`, which means a fixed issue disappears and a moved one shows up as new until you refresh.

### Warnings without blocking

During a gradual rollout you may want visibility without enforcement. Surface the count as a GitHub annotation instead of a failure:

```yaml
- run: eager_eye app/ --format json > report.json
- run: |
    issues=$(ruby -rjson -e 'puts JSON.parse(File.read("report.json"))["summary"]["total_issues"]')
    [ "$issues" -gt 0 ] && echo "::warning::Found $issues potential N+1 issues" || true
```

The [examples directory](https://github.com/hamzagedikkaya/eager_eye/blob/main/examples/github_action.yml) has a fuller version that turns each issue into a per-line `::warning file=...,line=...::` annotation on the PR diff.

---

## 7. RSpec Matcher and Auto-fix

### `pass_eager_eye`

If you'd rather enforce cleanliness from the test suite than from a separate CI step:

```ruby
# spec/rails_helper.rb
require "eager_eye/rspec"

# spec/eager_eye_spec.rb
RSpec.describe "EagerEye" do
  it "keeps controllers free of N+1 patterns" do
    expect("app/controllers").to pass_eager_eye
  end

  it "keeps serializers clean" do
    expect("app/serializers").to pass_eager_eye(only: [:serializer_nesting])
  end

  it "tolerates legacy code, for now" do
    expect("app/services/legacy").to pass_eager_eye(max_issues: 10)
  end
end
```

`only:`, `exclude:` and `max_issues:` cover the usual gradual-adoption needs. It runs in the same process as your specs and still never touches the database.

### Auto-fix (experimental)

```bash
eager_eye --suggest-fixes    # print diffs, change nothing
eager_eye --fix              # apply interactively
eager_eye --fix --force      # apply everything
```

| Finding | Rewrite |
|---|---|
| `.pluck(:id)` inside `.where(id: ...)` | `.select(:id)` |
| `.count` inside an iteration | `.size` |
| Association access in a loop with no preload | inserts `.includes(:assoc)` before the iteration |

The first two are mechanical and safe. The third is a *suggestion* — it can't know whether the collection is already preloaded three files away. Review the diff and run your suite.

---

## 8. VS Code Extension: Same Engine, in Your Editor

Running a CLI after every change is friction. The [EagerEye VS Code extension](https://marketplace.visualstudio.com/items?itemName=hamzagedikkaya.eager-eye) runs the gem on save and on file open and turns the JSON output into editor diagnostics:

- Squiggles at the offending line, tagged with the detector name, in the Problems panel
- Hover for the message and the suggested fix
- **Quick Fixes**: the `.pluck` → `.select` rewrite, plus one-click "disable for this line / this file" that inserts the gem's suppression comment for you
- Status bar: `EagerEye: OK` or `EagerEye: 3 issues`; click to re-run
- `EagerEye: Analyze Workspace` for a one-off pass over every Ruby file

It's a thin shell around the gem — `eager_eye <file> --format json --no-fail` — so new detectors appear in the editor without an extension update. Set `eagerEye.gemPath` to `bundle exec eager_eye` if the gem lives in your bundle rather than on the global path.

One limitation worth knowing: because the editor hands the gem a **single file**, the model-metadata pass doesn't run, so `DelegationNPlusOne`, `ScopeChainNPlusOne` and `ValidationNPlusOne` don't fire in the editor and association detection falls back to name heuristics. `db/schema.rb` *is* found, since the gem walks up from the file. The CLI on `app/` in CI gives you the full picture; the extension gives you the fast loop.

---

## 9. Suppressing False Positives

RuboCop-style comments, accepted in either `CamelCase` or `snake_case`:

```ruby
user.posts.count  # eager_eye:disable CountInIteration

# eager_eye:disable-next-line LoopAssociation
@users.each { |u| u.profile }

# eager_eye:disable LoopAssociation, SerializerNesting
@users.each { |u| u.posts.each { |p| p.author } }
# eager_eye:enable LoopAssociation, SerializerNesting

# eager_eye:disable-file custom_method_query     # must be in the first 5 lines

user.posts.count  # eager_eye:disable CountInIteration -- counter_cache handles this

# eager_eye:disable all
```

The `-- reason` suffix is documentation only, but reviewers will thank you. Suppressions are honoured identically by the CLI, the RSpec matcher and the editor, so a comment you add via Quick Fix carries over to CI.

---

## 10. Known Limitations

Static analysis isn't magic. What EagerEye can't do today:

**Cross-file flow.** Preload context propagates across methods in the same class, but not across files. When a controller does `OrderProcessor.new(orders).call` and the iteration lives in `OrderProcessor`, the analyzer can't see that `orders` was preloaded. Partials have the same problem. This is where most of the remaining false positives come from.

**Runtime facts.** It reads `db/schema.rb` for column names, but it doesn't know how many rows a table has, whether an index exists, or how hot an endpoint is. `Post.where(active: true).each` looks identical at 10 rows and 10 million. Bullet, `strict_loading`, and production monitoring cover that side.

**Heuristic association detection.** When neither the parsed models nor the schema can identify a receiver, EagerEye falls back to a short list of common association names (`author`, `user`, `posts`, …). Exotic naming can slip through; the list errs toward silence.

**Unparseable files are skipped, not analyzed.** A file the parser can't lex — a binary string literal with invalid UTF-8 escapes, an unknown `# encoding:` comment — is dropped with a single `EagerEye: Skipped unparseable file ...` line rather than crashing the run (1.3.2). The skipped list is available via `Analyzer#skipped_files`.

The honest summary hasn't changed: use EagerEye **alongside** Bullet, not instead of it. Static catches the code paths runtime never reaches; runtime catches what static can't see.

---

## 11. Roadmap and How to Contribute

Baseline mode was the top item here last time; it shipped in 1.3.1. What's next:

- **Inter-file call graph** — propagate preload context into `include`d modules and called service objects. This is the main remaining false-positive source, and the hardest item on the list.
- **Unified linter output** — [Reek](https://github.com/troessner/reek) / [RuboCop](https://rubocop.org/) style formatters so EagerEye findings sit in the same report as everything else.
- **Trend tracking** — a small dashboard over successive JSON reports, so a team can watch its N+1 count go down.

The gem is MIT-licensed, sits at roughly 95% test coverage, and welcomes PRs:

- Gem: [github.com/hamzagedikkaya/eager_eye](https://github.com/hamzagedikkaya/eager_eye)
- VS Code extension: [github.com/hamzagedikkaya/eager_eye_vscode](https://github.com/hamzagedikkaya/eager_eye_vscode)

---

## Conclusion

The [previous post]({% post_url 2025-12-06-beyond_n+1 %}) argued that Bullet only catches the tip of the N+1 iceberg. EagerEye is my attempt at the rest: automatic, on every PR, with no database, no fixtures, and no new DSL.

It won't catch everything — static analysis fundamentally can't — and the first published false-positive number in this post was wrong by an order of magnitude, which is its own lesson about sampling. But the tool shifts detection from "after deploy, when production pages you" to "before merge, when the fix is one `.includes` away", and baseline mode means you can turn it on today without fixing a thousand legacy warnings first.

If you run it on a real codebase and find something — a real N+1 or a false positive — open an issue. The tool got better every time someone did.

---

## Resources

- [EagerEye on RubyGems](https://rubygems.org/gems/eager_eye) · [GitHub](https://github.com/hamzagedikkaya/eager_eye) · [CHANGELOG](https://github.com/hamzagedikkaya/eager_eye/blob/main/CHANGELOG.md)
- [VS Code extension](https://marketplace.visualstudio.com/items?itemName=hamzagedikkaya.eager-eye)
- [Beyond N+1: Hidden Performance Traps and Fixes]({% post_url 2025-12-06-beyond_n+1 %}) — the post that motivated this tool
- [Meridian]({% post_url 2026-05-23-meridian %}) — the Rails 8 app that runs EagerEye in its dev Gemfile
- [Bullet](https://github.com/flyerhzm/bullet) · [Prosopite](https://github.com/charkost/prosopite)
- [whitequark/parser](https://github.com/whitequark/parser) — the AST library EagerEye is built on
- [RuboCop](https://github.com/rubocop/rubocop) — inspiration for the suppression syntax
