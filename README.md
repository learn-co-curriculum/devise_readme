# Devise

## Learning Objectives

1. Describe the major architecture and modules of Devise.
2. Build a working login system using Devise.

## Overview

[Devise] is a gem for when you have a lot of authentication needs. Common
security-based tasks that you know from internet site use are all provided:

* Sign-up confirmation emails
* Locking accounts after failed password attempts
* Emailing password resets
* Role-based permission granting (e.g. an "admin" can view this page, but a
  "manager" cannot)

Devise also has an extensive collection of generators that create controllers
and views. We can run one short command and Devise will set up a login page for
us. There's only one catch: because so much happens without our having to
understand how it works, Devise is effectively doing magic. Creating
customization or debugging Devise-generated shortcuts becomes a lot harder.

In this document we'll describe the architecture of [Devise] and list some of
the shortcuts it provides.

## Architecture

Devise is a Rails [engine]. That means it's basically a Rails app that sits
inside your Rails app. It has its own views and controllers and defines its own
routes.

![rails-xzibit](https://user-images.githubusercontent.com/16994388/51770489-94d51600-20ab-11e9-9581-0328bf626cef.jpg)

It does not define models for you, but it does have generators (short
command-line scripts that do multiple updates to a Rails application e.g.
`rails generate devise:install`) that make the process of creating a
Devise-compliant `User` model very easy.

Devise is made up of modules. Modules are applied to your `User` model, so you
should read these as abilities that `User` accounts can have:

* **Database Authenticatable:** encrypts and stores a password in the database to validate the authenticity of a user while signing in. The authentication can be done both through `POST` requests or HTTP Basic Authentication.
* **Omniauthable:** adds [OmniAuth](https://github.com/intridea/omniauth) support.
* **Confirmable:** sends emails with confirmation instructions and verifies whether an account is already confirmed during sign in.
* **Recoverable:** resets the user password and sends reset instructions.
* **Registerable:** handles signing up users through a registration process, also allowing them to edit and destroy their account.
* **Rememberable:** manages generating and clearing a token for remembering the user from a saved cookie.
* **Trackable:** tracks sign in count, timestamps and IP address.
* **Timeoutable:** expires sessions that have not been active in a specified period of time.
* **Validatable:** provides validations of email and password. It's optional and can be customized, so you're able to define your own validations.
* **Lockable:** locks an account after a specified number of failed sign-in attempts. Can unlock via email or after a specified time period.

That's all from the [Devise docs][Devise].

## Configuration

Devise is most often installed in order to add features to a `User` model.
Several of the modules come added for you by default if you run `rails generate
devise User`. This generates a `User` model that includes some helpful notes at
the top:

```ruby
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable
end
```

That's Devise wiring itself into the model. It also creates a migration for all
the fields it needs.

Let's look at the highlights.

### [database_authenticable]

This adds a `valid_password?(password)` method to the model. The password is
stored securely in the database.

### [registerable]

Registerable gives you `User.new_with_session(params, session)`, which lets you
initialize a `User` from session data (like a token from Facebook) in addition
to params.

### [recoverable]

Recoverable gives you password resets, like so:

```ruby
# resets the user password and save the record, true if valid passwords are given, otherwise false
User.find(1).reset_password('password123', 'password123')

# only resets the user password, without saving the record
user = User.find(1)
user.reset_password('password123', 'password123')

# creates a new token and sends it with instructions about how to reset the password
# (this one requires a mailer.)
User.find(1).send_reset_password_instructions
```

### [rememberable]

This lets you remember a user and associate them with a `User` object in the database without them having to log in. It works by storing a token in cookies.

```ruby
User.find(1).remember_me!  # regenerating the token
User.find(1).forget_me!    # clearing the token
```

### [trackable]

Track information about your user's sign-ins. It tracks the following columns:
* `sign_in_count` — Increased every time a user signs in (by form, OpenID, OAuth, etc.)
* `current_sign_in_at` — A timestamp updated when the user signs in
* `last_sign_in_at` — Holds the timestamp of the previous sign-in
* `current_sign_in_ip` — The remote IP updated when the user signs in
* `last_sign_in_ip` — Holds the remote IP of the previous sign-in

### [validatable]

The documentation on this is quite clear:

>Validatable creates all needed validations for a user email and password. It's
>optional, given you may want to create the validations by yourself.
>Automatically validate if the email is present, unique and its format is
>valid. Also tests presence of password, confirmation and length.

### [lockable]

Handles blocking a user's access after a certain number of attempts. Lockable
accepts two different strategies to unlock a user after they're blocked: email
and time. The former will send an email to the user when the lock happens,
containing a link to unlock their account. The second will unlock the user
automatically after some configured time (e.g., two hours). It's also possible
to set up Lockable to use both email and time strategies.

### [omniauthable]

This one doesn't give you a whole lot more than OmniAuth already does. It does
set some (but not all!) of the routes for you. That's a nice touch.

## Typical Setup

Use the `rails` command to generate a new application.

Add Devise to your Gemfile:
```ruby
gem 'devise'
```

Run `bundle install`.

Now run the installer:
```ruby
rails generate devise:install
```

Devise then tells us what we need to do. We'll go through these steps below,
but since it's so "magical," Devise has to prompt us as to _exactly_ what
we need to do. We'll include the output here, for reference.

```text
Some setup you must do manually if you haven't yet:

  1. Ensure you have defined default url options in your environments files. Here
     is an example of default_url_options appropriate for a development environment
     in config/environments/development.rb:

       config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }

     In production, :host should be set to the actual host of your application.

  2. Ensure you have defined root_url to *something* in your config/routes.rb.
     For example:

       root to: "home#index"

  3. Ensure you have flash messages in app/views/layouts/application.html.erb.
     For example:

       <p class="notice"><%= notice %></p>
       <p class="alert"><%= alert %></p>

  4. You can copy Devise views (for customization) to your app by running:

       rails g devise:views
```

In summary it says:

1. Make you sure you set up Rails to be able to send email (for password
   lock-out, etc.)
2. Make sure you have a route
3. Make sure you have flash messages being displayed so that Devise feedback
   (password expired, password wrong, etc.) can be shown
4. Here's how to customize views (take a little bit of "magic" out).

Our insructions will only cover steps 2 and 3.

The `generate devise:install` command gives us a ***very important*** file:
`config/initializers/devise.rb`. This is probably the single best source of
documentation for Devise. When you need to start debugging or theorizing about
the magic (maybe doing some customization), start here.

## Obeying the Installer: Setting up Default Routes

The installer advised us to create a route, we're going to do it.

Create a `WelcomeController` to correspond with this route. Give it a `#home`
action with view.

The `config/routes.rb` file should look like:

```ruby
Rails.application.routes.draw do
  get 'welcome/home'
  root to: 'welcome#home'
end
```

Ensure your `http://localhost:3000` routes you to your `WelcomeController`'s
`home` action.

![User sign_in home page](https://curriculum-content.s3.amazonaws.com/web-development/rails/devise_readme/devise_login.png)

Stop your server after verifying your root route works.

## Creating a Devise-Managed `User` Model

Now generate your `User` model with:

```shell
rails generate devise User # Also works: rails g devise User 
```

Run `rake routes`. See all the new routes beyond our mere `welcome#home`?

Run `rails db:migrate` to build up the database table holding our `users`.

Run `rails s` to start the server again and take a look at one of the
Devise-generated routes like `/users/sign_in`.

You should now have a working app with sign-in/sign-in capability!

With this work done, we essentially have the value of Devise. The next steps
are all about making navigation to these routes feel natural and making Devise
feedback visible to the user.

## Integrating Devise

If you look at the routes you can see that Devise gives us a `sign_out` route
as well, but visitors won't know to type that URL. Let's implement that view.

We probably want the user to be able to click 'Sign Out' on any page when
they're logged in, so let's add that to our layout, the common template shared
throughout the application. In the code below we also use the Devise-provided
method `user_signed_in?`.

```erb
<!-- views/layouts/application.html.erb -->

  ...

    <% if user_signed_in? %>
      <%= link_to("Sign Out", destroy_user_session_path, :method => 'delete') %>
    <% else %>
      <%= link_to("Sign In", new_user_session_path) %>
    <% end %>

  ...
```

## Obeying the Installer: Showing Messages in the `flash`

Devise will also add messages to the `flash` when a user signs in or when
there's an error. The [`flash`][flash] is a message that appears when a view is
rendered, but it only sticks around one page load.

Visit our "Sign In" path and enter a valid use with an invalid password. See,
how there's no feedback? We want users to see these errors.

> **PRO-TIP**: Keep in mind, by looking at the Rails log developers can see
> what happened:
> 
> Started POST "/users/sign_in" for ::1 at 2019-10-16 08:48:47 -0400
> Processing by Devise::SessionsController#create as HTML
> ...
> Completed 401 Unauthorized in 182ms (ActiveRecord: 0.8ms)

```erb
<!-- views/layouts/application.html.erb -->

  ...

  <%= content_tag(:div, flash[:error], :id => "flash_error") if flash[:error] %>
  <%= content_tag(:div, flash[:notice], :id => "flash_notice") if flash[:notice] %>
  <%= content_tag(:div, flash[:alert], :id => "flash_alert") if flash[:alert] %>

  ...
```

While it won't win any beauty awards, it communicates what happened.

![Invalid login](https://curriculum-content.s3.amazonaws.com/web-development/rails/devise_readme/devise_with_flash.png)

## Conclusion

In a real app, we'd probably want to add some CSS and probably put these pieces
into partials, like a `header` or `nav` partial, but you can see how quickly
and with how little code we were able to get a fully functioning login system.
Devise is a substantial gem. It offers many capabilities. Most developers are
able to get the basic login working in minutes, but to build out a full
authentication model will take hours to days.

A common pattern is to develop a site with Devise and then remove it later,
replacing only the pieces that you need _by hand_ so that the "magic" is gone
from the system. Maybe simple authentication by-hand is sufficient, maybe
another application needs more. There are no set rules here so feel free to
decide what works _for your application_.

## Resources

* [How to use Devise](https://launchschool.com/blog/how-to-use-devise-in-rails-for-authentication)
* [Devise][Devise]
* [engine][engine]
* [registerable], [database_authenticable], [recoverable], [rememberable], [trackable], [validatable], [lockable], [omniauthable]

[Devise]: https://github.com/plataformatec/devise
[engine]: http://guides.rubyonrails.org/engines.html
[registerable]: http://www.rubydoc.info/github/plataformatec/devise/master/Devise/Models/Registerable
[database_authenticable]: http://www.rubydoc.info/github/plataformatec/devise/master/Devise/Models/DatabaseAuthenticatable
[recoverable]: http://www.rubydoc.info/github/plataformatec/devise/master/Devise/Models/Recoverable
[rememberable]: http://www.rubydoc.info/github/plataformatec/devise/master/Devise/Models/Rememberable
[trackable]: http://www.rubydoc.info/github/plataformatec/devise/master/Devise/Models/Trackable
[validatable]: http://www.rubydoc.info/github/plataformatec/devise/master/Devise/Models/Validatable
[lockable]: http://www.rubydoc.info/github/plataformatec/devise/master/Devise/Models/Lockable
[omniauthable]: http://www.rubydoc.info/github/plataformatec/devise/master/Devise/Models/Omniauthable

<p class='util--hide'>View <a href='https://learn.co/lessons/devise_readme'>Devise</a> on Learn.co and start learning to code for free.</p>
[flash]: https://api.rubyonrails.org/classes/ActionDispatch/Flash.html
