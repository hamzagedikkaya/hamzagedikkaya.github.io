---
layout: post
title: "Modern Rails Authentication with Devise & Hotwire"
author: Hamza Gedikkaya
categories: 
  - Rails Projects
excerpt_image: /assets/images/posts/second_gif.gif
banner:
  image: /assets/images/posts/second_post.jpg
  height: "50vh"
tags: 
  - Ruby on Rails
  - Devise
  - Hotwire
  - Authentication
---

Building robust user authentication is a fundamental requirement for most web applications. In this guide, we'll walk through setting up a complete authentication system in Rails using Devise, integrating it seamlessly with Hotwire for a modern SPA-like experience, enhancing our forms with Simple Form, and implementing user profile images with Active Storage.

By the end of this guide, you'll have a fully functional authentication system that handles user registration, login, profile management, and image uploads—all working smoothly with Turbo.

---

## Table of Contents

1. [Devise Setup](#1-devise-setup)
2. [Hotwire Integration](#2-hotwire-integration)
3. [Simple Form Configuration](#3-simple-form-configuration)
4. [Active Storage & Image Processing](#4-active-storage--image-processing)

---

## 1. Devise Setup

Devise is the de facto standard for authentication in Rails applications. It provides a complete MVC solution with modules for password recovery, session management, email confirmation, and more.

### Installation

```bash
bundle add devise
rails generate devise:install
rails generate devise:views
rails generate devise User
```

After running the generator, Devise will create a migration file in `db/migrate/`. Before running the migration, let's add some custom fields to our User model.

### Adding Custom Fields

Open the generated migration file and add the following fields within the `create_table` block:

```ruby
# db/migrate/XXXXXX_devise_create_users.rb

def change
  create_table :users do |t|
    ## Database authenticatable
    t.string :email,              null: false, default: ""
    t.string :encrypted_password, null: false, default: ""

    # ... other Devise fields ...

    ## Custom Fields
    t.string :name_surname, null: false, default: ""
    t.string :gsm
    t.date   :date_of_birth

    t.timestamps null: false
  end

  add_index :users, :email, unique: true
  add_index :users, :reset_password_token, unique: true
end
```

### Configuring Strong Parameters

When adding custom fields, we need to permit them in Devise's strong parameters. First, update your routes to use a custom registrations controller:

```ruby
# config/routes.rb

Rails.application.routes.draw do
  devise_for :users, controllers: { registrations: "users/registrations" }
  
  # ... other routes ...
end
```

Then create the custom controller:

```ruby
# app/controllers/users/registrations_controller.rb

class Users::RegistrationsController < Devise::RegistrationsController
  before_action :configure_sign_up_params, only: [:create]
  before_action :configure_account_update_params, only: [:update]

  protected

  def configure_sign_up_params
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name_surname, :gsm, :date_of_birth])
  end

  def configure_account_update_params
    devise_parameter_sanitizer.permit(:account_update, keys: [:name_surname, :gsm, :date_of_birth])
  end
end
```

Now run the migration:

```bash
rails db:migrate
```

---

## 2. Hotwire Integration

Hotwire (HTML Over The Wire) is Rails' answer to building reactive applications without writing custom JavaScript. However, Devise was built before Hotwire existed, so we need to make a few adjustments to ensure they work together smoothly.

### The Problem

By default, when Devise encounters an authentication error (invalid credentials, unauthorized access, etc.), it responds with HTTP status codes that Turbo doesn't handle gracefully. This can result in broken redirects or missing flash messages.

### Creating a Custom Failure App

To handle authentication failures properly with Turbo, create a custom failure app:

```ruby
# lib/turbo_failure_app.rb

class TurboFailureApp < Devise::FailureApp
  def respond
    if request_format == :turbo_stream
      redirect
    else
      super
    end
  end

  def skip_format?
    %w[html turbo_stream */*].include?(request_format.to_s)
  end
end
```

### Configuring Devise for Turbo

Update your Devise initializer to use the custom failure app:

```ruby
# config/initializers/devise.rb

# Ensure the custom failure app is loaded
require "turbo_failure_app"

Devise.setup do |config|
  # ... other configurations ...

  # Add turbo_stream to navigational formats
  config.navigational_formats = ["*/*", :html, :turbo_stream]

  # Configure Warden to use our custom failure app
  config.warden do |manager|
    manager.failure_app = TurboFailureApp
  end
end
```

### How It Works

| Component | Purpose |
|-----------|---------|
| `TurboFailureApp` | Intercepts authentication failures and ensures proper redirect behavior for Turbo Stream requests |
| `skip_format?` | Allows the failure app to handle HTML, Turbo Stream, and wildcard formats |
| `navigational_formats` | Tells Devise which response formats should trigger redirects instead of 401 responses |

With this configuration, your Devise authentication will work seamlessly with Turbo Drive and Turbo Frames.

---

## 3. Simple Form Configuration

Simple Form is a powerful form builder that reduces boilerplate and integrates beautifully with CSS frameworks like Bootstrap and Tailwind.

### Installation

```bash
bundle add simple_form
rails generate simple_form:install

# For Bootstrap projects:
rails generate simple_form:install --bootstrap
```

### Updating Devise Views

Let's refactor the Devise login page to use Simple Form with Tailwind CSS styling:

```erb
<%# app/views/devise/sessions/new.html.erb %>

<div class="max-w-md mx-auto bg-white shadow-lg rounded-lg p-8 border border-gray-300">
  <h2 class="text-3xl font-bold text-center mb-8 text-gray-800">Log in</h2>

  <%= simple_form_for(resource, as: resource_name, url: session_path(resource_name)) do |f| %>
    <div class="space-y-4">
      <%= f.input :email,
                  label: "Email",
                  required: true,
                  autofocus: true,
                  input_html: {
                    autocomplete: "email",
                    class: "mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"
                  } %>

      <%= f.input :password,
                  label: "Password",
                  required: true,
                  input_html: {
                    autocomplete: "current-password",
                    class: "mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"
                  } %>

      <% if devise_mapping.rememberable? %>
        <%= f.input :remember_me,
                    as: :boolean,
                    label: "Remember me",
                    wrapper_html: { class: "flex items-center" },
                    input_html: { class: "h-4 w-4 text-blue-600 border-gray-300 rounded" } %>
      <% end %>

      <%= f.button :submit,
                   "Log in",
                   class: "w-full py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 transition duration-150" %>
    </div>
  <% end %>

  <div class="mt-6 text-center text-sm text-gray-600">
    <%= render "devise/shared/links" %>
  </div>
</div>
```

---

## 4. Active Storage & Image Processing

Active Storage provides a simple way to attach files to Active Record models. Combined with the `image_processing` gem, we can handle user profile images with validation and transformations.

### Installation

First, uncomment the `image_processing` gem in your Gemfile:

```ruby
# Gemfile
gem "image_processing", "~> 1.2"
```

Then install and set up Active Storage:

```bash
bundle install
rails active_storage:install
rails db:migrate
```

### Attaching Images to Users

Update the User model to accept profile images:

```ruby
# app/models/user.rb

class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  has_one_attached :profile_image

  validate :acceptable_image

  private

  def acceptable_image
    return unless profile_image.attached?

    # Validate file size (max 10MB)
    if profile_image.byte_size > 10.megabytes
      errors.add(:profile_image, I18n.t("errors.messages.profile_image_too_large", default: "is too large (maximum is 10MB)"))
    end

    # Validate content type
    acceptable_types = ["image/jpeg", "image/jpg", "image/png", "image/webp"]
    unless acceptable_types.include?(profile_image.content_type)
      errors.add(:profile_image, I18n.t("errors.messages.profile_image_invalid_format", default: "must be a JPEG, PNG, or WebP image"))
    end
  end
end
```

### Updating the Registration Form

Add the file input to your edit registration view:

```erb
<%# app/views/devise/registrations/edit.html.erb %>

<%= simple_form_for(resource, as: resource_name, url: registration_path(resource_name), html: { method: :put, multipart: true }) do |f| %>
  
  <%# ... other fields ... %>

  <div class="space-y-2">
    <% if resource.profile_image.attached? %>
      <div class="mb-4">
        <%= image_tag resource.profile_image.variant(resize_to_limit: [150, 150]),
                      class: "rounded-full border-2 border-gray-200" %>
      </div>
    <% end %>

    <%= f.input :profile_image,
                as: :file,
                label: "Profile Image",
                hint: "Accepted formats: JPEG, PNG, WebP. Maximum size: 10MB",
                input_html: {
                  accept: "image/jpeg,image/png,image/jpg,image/webp",
                  class: "block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-md file:border-0 file:text-sm file:font-semibold file:bg-blue-50 file:text-blue-700 hover:file:bg-blue-100"
                } %>
  </div>

  <%# ... submit button ... %>
<% end %>
```

### Updating Strong Parameters

Don't forget to permit the `profile_image` parameter in your registrations controller:

```ruby
# app/controllers/users/registrations_controller.rb

def configure_account_update_params
  devise_parameter_sanitizer.permit(:account_update, keys: [:name_surname, :gsm, :date_of_birth, :profile_image])
end
```

---

## Conclusion

We've built a complete, modern authentication system that combines the reliability of Devise with the reactivity of Hotwire. Here's what we accomplished:

- **Devise**: Handles all authentication logic with custom user fields
- **Hotwire**: Provides seamless page updates without full reloads
- **Simple Form**: Creates clean, maintainable forms with minimal code
- **Active Storage**: Manages user profile images with proper validation

This setup provides a solid foundation that you can extend with additional features like OAuth providers, two-factor authentication, or email confirmation as your application grows.

---

## Resources

- [Devise Documentation](https://github.com/heartcombo/devise)
- [Hotwire Documentation](https://hotwired.dev/)
- [Simple Form Documentation](https://github.com/heartcombo/simple_form)
- [Active Storage Guide](https://guides.rubyonrails.org/active_storage_overview.html)
