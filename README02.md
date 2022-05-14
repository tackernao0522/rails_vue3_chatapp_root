# CORSの設定

+ `api/Gemfile`を編集<br>

```:Gemfile
source 'https://rubygems.org'
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby '2.6.4'

# Bundle edge Rails instead: gem 'rails', github: 'rails/rails', branch: 'main'
gem 'rails', '~> 6.0.5'
# Use postgresql as the database for Active Record
gem 'pg', '>= 0.18', '< 2.0'
# Use Puma as the app server
gem 'puma', '~> 4.1'
# Build JSON APIs with ease. Read more: https://github.com/rails/jbuilder
# gem 'jbuilder', '~> 2.7'
# Use Redis adapter to run Action Cable in production
# gem 'redis', '~> 4.0'
# Use Active Model has_secure_password
# gem 'bcrypt', '~> 3.1.7'

# Use Active Storage variant
# gem 'image_processing', '~> 1.2'

# Reduces boot times through caching; required in config/boot.rb
gem 'bootsnap', '>= 1.4.2', require: false

# Use Rack CORS for handling Cross-Origin Resource Sharing (CORS), making cross-origin AJAX possible
gem 'rack-cors' # コメントアウト解除

gem 'devise', '~> 4.8'
gem 'devise_token_auth', '~> 1.1', '>= 1.1.5'

group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger console
  gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]
end

group :development do
  gem 'listen', '~> 3.2'
  # Spring speeds up development by keeping your application running in the background. Read more: https://github.com/rails/spring
  gem 'spring'
  gem 'spring-watcher-listen', '~> 2.0.0'
end

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]
```

+ `root $ docker compose buid api`を実行<br>

+ `root $ root $ docker compose run --rm api bundle info rack-cors`を実行して入ったか確認してみる<br>

+ `api/config/initialize/cors.rb`を編集<br>

```rb:cors.rb
# Be sure to restart your server when you modify this file.

# Avoid CORS issues when API is called from the frontend app.
# Handle Cross-Origin Resource Sharing (CORS) in order to accept cross-origin AJAX requests.

# Read more: https://github.com/cyu/rack-cors

Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV['API_DOMAIN'] || ''

    resource '*',
      headers: :any,
      expose: ['access-token', 'expiry', 'token-type', 'uid', 'client'],

      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

## ルーティングの設定とコントローラの作成と設定

+ `config/routes.rb`を編集<br>

```rb:routes.rb
Rails.application.routes.draw do
  mount_devise_token_auth_for 'User', at: 'auth', controllers: {
    registrations: 'auth/registrations'
  }
end
```

+ `root $ docker compose run --rm api rails g controller auth/registrations`を実行<br>

+ `api/app/controllers/auth/registrations_controller.rb`を編集<br>

```rb:registrations_controller.rb
class Auth::RegistrationsController < ApplicationController

  private

  def sign_up_params
    params.permit(:name, :email, :password, :password_confirmation)
  end
end
```
