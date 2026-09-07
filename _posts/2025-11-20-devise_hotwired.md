---
layout: post
title: "Modern Rails Authentication with Devise & Hotwire"
author: Hamza Gedikkaya
categories: 
  - Rails Projects
excerpt_image: /assets/images/posts/first_gif.gif
banner:
  image: /assets/images/posts/first_post.jpg
  height: "50vh"
tags: 
  - Ruby on Rails
  - Devise
  - Hotwire
  - Authentication
---

Authentication is the first thing almost every Rails app needs and the last thing you want to get subtly wrong. This guide walks through a complete, production-shaped setup: **Devise** for the authentication core, **Hotwire** (Turbo + Stimulus) for a snappy, no-custom-JavaScript frontend, **Simple Form** for forms that don't drown in markup, and **Active Storage** for profile images with real validation.

By the end you'll have registration, login, profile editing, and avatar uploads — all working *with* Turbo instead of fighting it.

> *Updated September 2026.* The original version of this post worked around Devise's Turbo incompatibility with a custom failure app. Devise 4.9+ made that unnecessary, and this revision reflects the modern two-line configuration. The legacy approach is kept in a short aside for anyone maintaining an older codebase.

---

## Table of Contents

1. [Devise or the Rails 8 Authentication Generator?](#1-devise-or-the-rails-8-authentication-generator)
2. [Devise Setup with Custom Fields](#2-devise-setup-with-custom-fields)
3. [Making Devise Speak Turbo](#3-making-devise-speak-turbo)
4. [Simple Form Without the Boilerplate](#4-simple-form-without-the-boilerplate)
5. [Profile Images with Active Storage](#5-profile-images-with-active-storage)
6. [The Same Stack in a Real App](#6-the-same-stack-in-a-real-app)

---

## 1. Devise or the Rails 8 Authentication Generator?

Rails 8 ships `bin/rails generate authentication`, which scaffolds a minimal, readable session + password-reset flow with no dependencies. It's a great starting point if you want to own every line of your auth code.

Devise is the other answer. It gives you, out of the box, the pieces you'd otherwise rebuild one by one: email confirmation, account locking, remember-me cookies, session timeouts, sign-in tracking, OmniAuth hooks, and a decade of security patches behind each of them.

| | Rails 8 generator | Devise |
|---|---|---|
| Dependencies | none | one gem |
| Password reset | yes | yes (`:recoverable`) |
| Email confirmation | build it | `:confirmable` |
| Account lockout | build it | `:lockable` |
| OAuth providers | build it | `:omniauthable` |
| Session timeout | build it | `:timeoutable` |
| Customization | edit the generated code | override controllers / views |

My rule of thumb: prototypes and single-purpose apps get the generator, anything that will have real users gets Devise. This guide uses Devise 5.

---

## 2. Devise Setup with Custom Fields

### Installation

```bash
bundle add devise
bin/rails generate devise:install
bin/rails generate devise:views
bin/rails generate devise User
```

The install generator prints a checklist. The one item people skip is the mailer host, and password resets silently break without it:

```ruby
# config/environments/development.rb
config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
```

### Adding Custom Fields

Before running the migration, open the generated `db/migrate/*_devise_create_users.rb` and add your own columns inside `create_table`:

```ruby
create_table :users do |t|
  ## Database authenticatable
  t.string :email,              null: false, default: ""
  t.string :encrypted_password, null: false, default: ""

  # ... other Devise fields ...

  ## Custom fields
  t.string :name_surname, null: false, default: ""
  t.string :gsm
  t.date   :date_of_birth

  t.timestamps null: false
end

add_index :users, :email,                unique: true
add_index :users, :reset_password_token, unique: true
```

```bash
bin/rails db:migrate
```

### Permitting the Custom Fields

Devise filters params through its own sanitizer, so new fields need to be permitted for `sign_up` and `account_update`. Point the routes at a custom registrations controller:

```ruby
# config/routes.rb
devise_for :users, controllers: { registrations: "users/registrations" }
```

```ruby
# app/controllers/users/registrations_controller.rb
class Users::RegistrationsController < Devise::RegistrationsController
  before_action :configure_sign_up_params,        only: :create
  before_action :configure_account_update_params, only: :update

  private

  PROFILE_KEYS = %i[name_surname gsm date_of_birth].freeze

  def configure_sign_up_params
    devise_parameter_sanitizer.permit(:sign_up, keys: PROFILE_KEYS)
  end

  def configure_account_update_params
    devise_parameter_sanitizer.permit(:account_update, keys: PROFILE_KEYS)
  end
end
```

Keeping the keys in one constant means adding a field later is a one-line change.

---

## 3. Making Devise Speak Turbo

### Why it broke in the first place

Turbo has two opinions about HTTP responses that Devise, written years earlier, didn't share:

1. **A form that fails validation must return `422 Unprocessable Content`**, not `200 OK`. Turbo treats a 200 from a form submission as "nothing to render" and silently ignores the body — so your error messages never appear.
2. **A redirect after a non-GET request should be `303 See Other`**, so Turbo follows it with a GET instead of replaying the original method.

Devise's default responder returned `200` on failed sign-in and `302` on redirects. The result was the classic symptom: submit a wrong password, nothing happens.

### The fix: two lines (Devise ≥ 4.9)

Devise 4.9 introduced a configurable responder. Set both statuses in the initializer and you're done:

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  # ...

  # Turbo expects 422 for re-rendered forms and 303 for redirects after POST/PATCH/DELETE.
  config.responder.error_status    = :unprocessable_content
  config.responder.redirect_status = :see_other
end
```

On Rails < 7.1 the status symbol is `:unprocessable_entity`; Rack 3.1 renamed it to `:unprocessable_content` and the old name now logs a deprecation warning.

You do **not** need to add `:turbo_stream` to `navigational_formats` unless you're rendering `turbo_stream` responses from Devise actions yourself. The default `['*/*', :html]` handles regular Turbo Drive navigation.

### Sign-out needs a button, not a link

`rails-ujs` is gone, so `link_to ..., method: :delete` no longer sends a DELETE. Use `button_to`:

```erb
<%= button_to "Sign out", destroy_user_session_path, method: :delete,
      class: "text-sm text-gray-600 hover:text-gray-900" %>
```

If you'd rather keep it a link, set `config.sign_out_via = :get` in the initializer — but a button is the safer default, since GET requests with side effects can be triggered by prefetching.

<details markdown="1">
<summary><strong>Legacy aside: the custom failure app (Devise &lt; 4.9)</strong></summary>

Before 4.9, the community workaround was to subclass `Devise::FailureApp` so Turbo Stream requests got a redirect instead of a 401:

```ruby
# lib/turbo_failure_app.rb
class TurboFailureApp < Devise::FailureApp
  def respond
    request_format == :turbo_stream ? redirect : super
  end

  def skip_format?
    %w[html turbo_stream */*].include?(request_format.to_s)
  end
end

# config/initializers/devise.rb
config.navigational_formats = ["*/*", :html, :turbo_stream]
config.warden { |manager| manager.failure_app = TurboFailureApp }
```

If you find this in a codebase you maintain, you can delete it once you're on Devise 4.9 or later and set the two responder statuses above instead.
</details>

---

## 4. Simple Form Without the Boilerplate

Simple Form wraps label, input, hint, and error rendering into one `f.input` call. Install it and, if you use Tailwind, skip the Bootstrap flag:

```bash
bundle add simple_form
bin/rails generate simple_form:install
```

### Put the CSS classes in the wrapper, not the view

The most common Simple Form mistake with Tailwind is pasting a 200-character class string into every input. Define a wrapper once instead:

```ruby
# config/initializers/simple_form_tailwind.rb
SimpleForm.setup do |config|
  config.wrappers :tailwind, tag: "div", class: "space-y-1", error_class: "has-error" do |b|
    b.use :html5
    b.use :placeholder
    b.optional :maxlength
    b.optional :pattern
    b.optional :readonly

    b.use :label, class: "block text-sm font-medium text-gray-700"
    b.use :input,
          class: "mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 shadow-sm " \
                 "focus:border-blue-500 focus:ring-blue-500",
          error_class: "border-red-500"
    b.use :hint,  wrap_with: { tag: "p", class: "text-xs text-gray-500" }
    b.use :error, wrap_with: { tag: "p", class: "text-xs text-red-600" }
  end

  config.default_wrapper = :tailwind
  config.button_class    = "w-full rounded-md bg-blue-600 px-4 py-2 text-sm font-medium text-white " \
                           "hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
end
```

### The login view, now readable

```erb
<%# app/views/devise/sessions/new.html.erb %>
<div class="mx-auto max-w-md rounded-lg border border-gray-200 bg-white p-8 shadow-lg">
  <h2 class="mb-8 text-center text-3xl font-bold text-gray-800">Log in</h2>

  <%= simple_form_for(resource, as: resource_name, url: session_path(resource_name)) do |f| %>
    <div class="space-y-4">
      <%= f.input :email,    required: true, autofocus: true,
                  input_html: { autocomplete: "email" } %>
      <%= f.input :password, required: true,
                  input_html: { autocomplete: "current-password" } %>

      <% if devise_mapping.rememberable? %>
        <%= f.input :remember_me, as: :boolean, wrapper: :default,
                    wrapper_html: { class: "flex items-center gap-2" } %>
      <% end %>

      <%= f.button :submit, "Log in" %>
    </div>
  <% end %>

  <div class="mt-6 text-center text-sm text-gray-600">
    <%= render "devise/shared/links" %>
  </div>
</div>
```

Same result as the class-per-input version, a third of the markup, and every form in the app looks consistent by default.

---

## 5. Profile Images with Active Storage

### Installation

```bash
bin/rails active_storage:install
bin/rails db:migrate
```

Add the image processing gems. Note that `image_processing` 2.x **no longer pulls in a processor** — you have to pick one explicitly:

```ruby
# Gemfile
gem "image_processing", "~> 2.0"
gem "ruby-vips"     # fast, low-memory; needs libvips installed
# or: gem "mini_magick"   # if ImageMagick is what your host already has
```

```ruby
# config/application.rb
config.active_storage.variant_processor = :vips   # or :mini_magick
```

### Attaching and validating

Active Storage doesn't ship attachment validations, so add your own — and use Rails 7's **named variants** so the thumbnail size is defined once, on the model, not scattered across views:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  ACCEPTED_IMAGE_TYPES = %w[image/jpeg image/png image/webp].freeze
  MAX_IMAGE_SIZE       = 10.megabytes

  has_one_attached :profile_image do |attachable|
    attachable.variant :thumb,  resize_to_limit: [150, 150]
    attachable.variant :avatar, resize_to_fill:  [40, 40]
  end

  validate :acceptable_profile_image

  private

  def acceptable_profile_image
    return unless profile_image.attached?

    if profile_image.byte_size > MAX_IMAGE_SIZE
      errors.add(:profile_image, :too_large, max: "10MB")
    end

    unless ACCEPTED_IMAGE_TYPES.include?(profile_image.content_type)
      errors.add(:profile_image, :invalid_format, formats: "JPEG, PNG or WebP")
    end
  end
end
```

```yaml
# config/locales/en.yml
en:
  activerecord:
    errors:
      models:
        user:
          attributes:
            profile_image:
              too_large: "is too large (maximum is %{max})"
              invalid_format: "must be a %{formats} image"
```

Using error *keys* instead of inline strings keeps the messages translatable — which matters the moment your app grows a second language.

### The edit form

```erb
<%# app/views/devise/registrations/edit.html.erb %>
<%= simple_form_for(resource, as: resource_name, url: registration_path(resource_name),
                    html: { method: :put, multipart: true }) do |f| %>

  <%# ... name, gsm, date_of_birth, email, passwords ... %>

  <div class="space-y-2">
    <% if resource.profile_image.attached? %>
      <%= image_tag resource.profile_image.variant(:thumb),
                    class: "mb-4 rounded-full border-2 border-gray-200" %>
    <% end %>

    <%= f.input :profile_image, as: :file,
                hint: "JPEG, PNG or WebP · max 10MB",
                input_html: {
                  accept: User::ACCEPTED_IMAGE_TYPES.join(","),
                  class:  "block w-full text-sm text-gray-500 file:mr-4 file:rounded-md " \
                          "file:border-0 file:bg-blue-50 file:px-4 file:py-2 file:text-blue-700"
                } %>
  </div>

  <%= f.button :submit, "Update" %>
<% end %>
```

And permit the new key:

```ruby
PROFILE_KEYS = %i[name_surname gsm date_of_birth profile_image].freeze
```

Because the form now returns `422` on a failed validation (thanks to §3), an oversized upload re-renders the form with the error message inline — no page reload, no lost input.

---

## 6. The Same Stack in a Real App

Everything above is exactly what [Meridian]({% post_url 2026-05-23-meridian %}) — the self-hosted life OS I'm building — runs on. A few details from that codebase that go one step further than this tutorial:

- **Preferences live on the user.** Timezone, currency, locale (Turkish / English), theme, and weekly-review day are Devise custom fields, permitted the same way as `name_surname` here. A multi-tab Settings page edits them.
- **Devise for the browser, bearer tokens for the phone.** The Flutter companion app doesn't use cookies. Each user has an `api_token`; the API controllers inherit from `ActionController::API` and authenticate with a one-line `Authorization: Bearer` check. Two auth paths, one `User` table.
- **No password-reset email.** It's a single-user, local-first app, so `:recoverable` stays on but SMTP is deliberately unconfigured. A forgotten password is reset from the Rails console — no phishing surface, no "link expired" friction.
- **`image_processing` + `mini_magick`,** declared explicitly, because the deployment host has ImageMagick but not libvips. That's the exact 2.x gotcha from §5, encountered in production.

---

## Conclusion

The modern Devise + Hotwire setup is a lot less code than it used to be:

- **Devise** — battle-tested authentication with a clean path for custom fields
- **Two responder statuses** — the entire Turbo fix, replacing the old failure-app subclass
- **Simple Form with a wrapper** — consistent forms without repeating class strings
- **Active Storage named variants** — thumbnails defined once, validated with translatable error keys

From here the natural extensions are `:confirmable` for email verification, `:omniauthable` for social login, and `devise-two-factor` for TOTP — each one a module flag away.

---

## Resources

- [Devise](https://github.com/heartcombo/devise) — see the "Hotwire/Turbo" section of the README
- [Hotwire](https://hotwired.dev/)
- [Simple Form](https://github.com/heartcombo/simple_form)
- [Active Storage Overview](https://guides.rubyonrails.org/active_storage_overview.html)
- [Rails 8 authentication generator](https://guides.rubyonrails.org/security.html#authentication)
- [Meridian]({% post_url 2026-05-23-meridian %}) — the app that runs this stack
