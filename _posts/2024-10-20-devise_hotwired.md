---
layout: post
title: Devise Hotwired
author: Hamza Gedikkaya
categories: 
  - Rails Projects
excerpt_image: /assets/images/posts/second_gif.gif
banner:
  image: /assets/images/posts/second_post.jpg
  height: "50vh"
tags: 
  - Leave Master Project
  - Gem
---

### Setting Up User Authentication with Devise

Devise is a popular gem for Ruby on Rails applications that simplifies the management of user authentication securely and efficiently. Below are the steps to install Devise and add custom fields to your user model.

```bash
# Add Devise gem to the Gemfile
bundle add devise

# Install Devise and generate necessary configuration files
rails generate devise:install

# Generate Devise views for user authentication
rails g devise:views

# Create User model with Devise
rails generate devise User
```

If you want to add custom fields to your user model, open the migration file located in the `db/migrate` directory and add the following code:

```ruby
## Custom Fields
t.string :name_surname, null: false, default: ""
t.string :gsm
t.date   :date_of_birth
```

If we add custom fields, we will need to follow a different approach to store this data within the user model.

**Define Routes**: Add the following line in `config/routes.rb`:
```ruby
devise_for :users, controllers: { registrations: "users/registrations" }
```
**Create a Registrations Controller**: We need to create a `registrations_controller.rb` file and update the `sign_up_params` and `account_update_params` methods.

You can find an example of this controller at the following link:  
[Registrations Controller - Leave Master](https://github.com/hamzagedikkaya/leave_master/blob/main/app/controllers/users/registrations_controller.rb)

To apply your changes, run the database migration:

```bash
rails db:migrate
```

---

## Hotwire

Hotwire is a framework for building modern web applications without relying heavily on JavaScript. It leverages server-side rendering and provides a seamless user experience by using Turbo and Stimulus. Below is an example of how to customize Devise to work with Turbo.

---

## Custom Failure App for Turbo

In order to handle authentication failures with Turbo Stream, you can create a custom failure app that inherits from `Devise::FailureApp`. Here’s how you can set it up:

```ruby
class TurboFailureApp < Devise::FailureApp
  def respond
    if request_format == :turbo_stream
      redirect
    else
      super
    end
  end

  def skip_format?
    %w[html turbo_stream */*].include? request_format.to_s
  end
end
```
Next, you need to configure Devise to use the custom `TurboFailureApp`. Open your `devise.rb` initializer and make the following changes:

```ruby
# Set navigational formats to include Turbo Stream
config.navigational_formats = [ "*/*", :html, :turbo_stream ]

config.warden do |manager|
  manager.failure_app = TurboFailureApp
  # manager.intercept_401 = false
  # manager.default_strategies(scope: :user).unshift :some_external_strategy
end
```

  - TurboFailureApp: This custom class overrides the respond method to check if the request format is turbo_stream. If it is, it redirects the user; otherwise, it falls back to the default behavior.
  - skip_format?: This method allows the app to skip the format check for certain formats, ensuring Turbo Stream responses are handled correctly.
  - navigational_formats: By including :turbo_stream, you ensure that Turbo can handle navigational responses.
  - warden configuration: This part sets the failure app to your custom class, allowing you to manage authentication failures more effectively.

When committing changes, a message like the following would be appropriate: `chore: setup Devise with Hotwire integration`. 

You can access the relevant commit [here](https://github.com/hamzagedikkaya/leave_master/commit/58b39158cfe6bbfecb032ed609cc714ab6c02f97).

---

## Simple Form Setup

**Simple Form** is a Rails form builder gem that simplifies form creation, providing a clean, user-friendly API for generating forms with minimal configuration. It also integrates well with popular CSS frameworks like Bootstrap.

```bash
# Add Simple Form to your project
bundle add simple_form

# Generate the Simple Form configuration file
rails generate simple_form:install

# Or, if you are using Bootstrap, you can use this command:
rails generate simple_form:install --bootstrap
```

**Apply Simple Form in Devise’s `sessions/new` Page:** Edit `app/views/devise/sessions/new.html.erb` to use simple_form_for, which will streamline the form's structure and style with minimal configuration.

```ruby
<div class="max-w-md mx-auto bg-white shadow-lg rounded-lg p-8 border border-gray-300">
  <h2 class="text-3xl font-bold text-center mb-8 text-gray-800">Log in</h2>

  <%= simple_form_for(resource, as: resource_name, url: session_path(resource_name)) do |f| %>
    <%= f.input :email, label: "Email", input_html: { autofocus: true, autocomplete: "email", class: "mt-2 block w-full border border-gray-300 rounded-lg shadow-sm focus:border-blue-500 focus:ring focus:ring-blue-500 focus:ring-opacity-50 p-2" }, wrapper_html: { class: "mb-4" } %>
    <%= f.input :password, label: "Password", input_html: { autocomplete: "current-password", class: "mt-2 block w-full border border-gray-300 rounded-lg shadow-sm focus:border-blue-500 focus:ring focus:ring-blue-500 focus:ring-opacity-50 p-2" }, wrapper_html: { class: "mb-6" } %>

    <% if devise_mapping.rememberable? %>
      <div class="flex items-center mb-4">
        <%= f.input :remember_me, as: :boolean, label: false, input_html: { class: "form-checkbox ml-2 h-4 w-4 text-blue-600 border-gray-300 rounded focus:ring-blue-500" } %>
        <label class="ml-2 text-sm text-gray-900"><%= f.label :remember_me, "Remember me" %></label>
      </div>
    <% end %>

    <div class="flex items-center justify-between mb-6 text-sm">
      <%= render "devise/shared/links" %>
    </div>

    <div>
      <%= f.submit "Log in", class: "w-full flex justify-center py-2 px-4 border border-transparent rounded-lg shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 transition duration-200" %>
    </div>
  <% end %>
</div>
```
The relevant commit for this setup can be found here: [feat: integrate Simple Form with Devise](https://github.com/hamzagedikkaya/leave_master/commit/947361d488ec150ea97176857e5f2b4aeded32e4)

---

## Image Processing Setup

To install the `image processing` gem, remove it from the comment in the `Gemfile` and then run the following command to install the gems:

```bash
bundle install

# After that, install Active Storage with the following command:
rails active_storage:install

# Finally, run the migrations:
rails db:migrate
```

After that, we can define a profile picture for users.

```ruby
# app/models/user.rb
has_one_attached :profile_image

# app/views/users/registrations/edit.html.erb
<%= f.input :profile_image, as: :file, label: "Profile Image", input_html: { accept: 'image/jpeg,image/png,image/jpg' } %>

# To display
<%= image_tag resource.profile_image if resource.profile_image.attached? %>
```

You can also add custom validation to ensure specific formats and size limits for the profile image. 
In your `user.rb` model, you can implement the following validation:

```ruby
validate :acceptable_image

private

def acceptable_image
  return unless profile_image.attached?

  errors.add(:profile_image, I18n.t("errors.messages.profile_image_too_large")) unless profile_image.byte_size <= 10.megabytes

  acceptable_types = ["image/jpg", "image/jpeg", "image/png"]
  return if acceptable_types.include?(profile_image.content_type)

  errors.add(:profile_image, I18n.t("errors.messages.profile_image_invalid_format"))
end
```

The relevant commit for this setup can be found here: [feat(profile): add image processing and user profile picture](https://github.com/hamzagedikkaya/leave_master/commit/3e3c00136fa2d81078328670948c7e9ddbefbb2b)

---
