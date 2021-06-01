---
title: "RailsのGemをいろいろ調べる"
date: 2021-06-01T23:00:36+09:00
draft: true
---

新しいプロジェクトにジョインしたのですが、知らないGemがたくさんあったので簡単にどんなGemがあるか知っておこうという話です。

```
# データの状態を簡単に管理する
gem 'aasm'

# View向けのモデルメソッドをデコレーターに移す
gem 'active_decorator'

# モデルを表に対応させる
gem 'active_importer'

# 大量データのインポートに使う
gem 'activerecord-import'

# マルチテナンシー？とやらを実現できるらしい
gem 'acts_as_tenant'

# ページネーションgemと組み合わせて使うなにか？
gem 'api-pagination'

# GraphQLでファイルをアップロード
gem 'apollo_upload_server'

# productionで使う？アセットをS3に同期
gem 'asset_sync'

# S3のAWS公式gem
gem 'aws-sdk-s3'

# SSMのAWS公式gem
gem 'aws-sdk-ssm'

# パスワードを暗号化
gem 'bcrypt'

# 営業日を考慮
gem 'business_time'

# エクセルを生成
gem 'caxlsx'

# caxlsxのテンプレートを提供
gem 'caxlsx_rails'


gem 'codecov'
gem 'coffee-rails'
gem 'counter_culture'
gem 'ddtrace'
gem 'doorkeeper'
gem 'doorkeeper-i18n'
gem 'devise'
gem 'devise-i18n'
gem 'devise-two-factor'
gem 'devise_invitable'
gem 'devise_saml_authenticatable'
gem 'enumerize'
gem 'faker'
gem 'figaro'
gem 'flag_shih_tzu'
gem 'fog-aws'
gem 'friendly_id'
gem 'fullcalendar-rails'
gem 'gimei'
gem 'globalize'
gem 'graphiql-rails'
gem 'graphql-batch'
gem 'groupdate'
gem 'holidays'
gem 'i18n-js'
gem 'jbuilder'
gem "jquery-fileupload-rails"
gem 'jquery-rails'
gem 'js-routes'
gem 'kaminari'
gem 'kaminari-i18n'
gem 'lockbox'
gem 'lograge'
gem 'logstash-event'
gem 'marcel'
gem 'meta-tags'
gem 'mimemagic'
gem 'mustache'
gem 'oauth2'
gem 'paranoia'
gem 'pg'
gem 'public_activity'
gem 'puma'
gem 'pundit'
gem 'rack-cors'
gem 'rack-maintenance'
gem 'rack-proxy'
gem 'rails-i18n'
gem 'ransack'
gem 'rbtrace'
gem 'redis'
gem 'redis-namespace'
gem 'redis-rails'
gem 'refile'
gem 'refile-mini_magick'
gem 'refile-s3'
gem 'remotipart'
gem 'responders'
gem 'rinku'
gem 'rolify'
gem 'rqrcode'
gem 'rubyzip'
gem 'sass-rails'
gem 'scenic'
gem 'secure_headers'
gem 'seed-fu'
gem 'sendgrid-actionmailer'
gem 'sentry-raven'
gem 'serviceworker-rails'
gem 'settingslogic'
gem 'shrine'
gem 'sidekiq'
gem 'simpacker'
gem 'simple_form'
gem 'sinatra'
gem 'slack-notifier'
gem 'slack-ruby-client'
gem 'slim-rails'
gem 'stackprof'
gem 'time_splitter'
gem 'timecop'
gem 'uglifier'
gem 'unf'
gem 'versioncake'
gem 'wareki'
gem 'webpush'
gem 'whenever'
gem 'wicked_pdf'
gem 'wkhtmltopdf-binary'
```
