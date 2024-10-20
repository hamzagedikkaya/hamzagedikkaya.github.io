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

To apply your changes, run the database migration:

```bash
rails db:migrate
```

---

## Hotwire

Hotwire is a framework for building modern web applications without relying heavily on JavaScript. It leverages server-side rendering and provides a seamless user experience by using Turbo and Stimulus. Below is an example of how to customize Devise to work with Turbo.

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
