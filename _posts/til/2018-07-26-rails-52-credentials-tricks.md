---
layout: til_post
title:  "Rails 5.2 credentials cheat cheat"
categories: til
disq_id: til-48
---

Rails introduced "encrypted" credentials from Rails version 5.2:

* <https://www.engineyard.com/blog/rails-encrypted-credentials-on-rails-5.2>
* <https://guides.rubyonrails.org/5_2_release_notes.html>

In order to use Rails credentials you need to have master key in
`config/master.key` or an environment variable `RAILS_MASTER_KEY`

In order to open the credentials file:

```ruby
EDITOR=vim rails credentials:edit

# or master key as env variable
RAILS_MASTER_KEY=xxxxxxxxxxxxxxxx EDITOR=vim rails credentials:edit
```

### Usage inside application code:

Fetch root value; e.g when credentials look like:

```
# ....
secret_key_base: yyyyyyyyyyyyyyyyyyyyyyyyy
# ....
```

```ruby
Rails.application.credentials.secret_key_base

Rails.application.credentials.fetch(:secret_key_base) { raise "it seems you didn't configure credentials" }

Rails.application.credentials[:secret_key_base] || "someDefaultValue"

ENV["SECRET_KEY_BASE"] ||  Rails.application.credentials[:secret_key_base]

Rails.application.credentials.dig :secret_key_base
```


fetch nested value; e.g credentials look like:

```
# ....
aws_s3:
  access_key: xxxxxxxxxxxxxxx

postgres:
  password:
    development: Yzx234354
    staging: "fooBar%@3"
# ....
```

```ruby
Rails.application.credentials.dig(:aws_s3, :access_key)

Rails.application.credentials
  .fetch(:aws_s3) { raise 'are you sure you have master key ?' }
  .fetch(:access_key) { 'someDefaultValue' }


pg_password = Rails.application.credentials.dig :postgres, :password, :development
pg_password = Rails.application.credentials.dig :postgres, :password, :staging

pg_password = Rails.application.credentials.dig(:postgres, :password, Rails.env.to_sym)
```

`config/database.yml` example:

```ruby
development:
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: postgres
  password: <%= ENV['RAILS_PG_PASS'] || Rails.application.credentials.dig(:postgres, :password, :development)
  host: <%= ENV['RAILS_PG_HOST'] || 'localhost' %>
  port: 5432
```


### More info:

```
rails credentials:help
```

### Security notes:

> Points here may seem obvious, but unfortunately I've already seen
> people doing these mistakes

#### Git

It is ok to commit `config/credentials.yml.enc` to git (that is its
purpose)

* Never commit `config/master.key` to git!
* Never commit value of `RAILS_MASTER_KEY` to git!

If you did commit them at any point in the past, erase the git commits from git history or much better
regenerate the master.key (section bellow **Regenerate key**)

Make sure `config/master.key` is in your `.gitignore`. This apply for any file
that reference `RAILS_MASTER_KEY` environment variable.

#### Docker

Make sure `config/master.key` is in your `.dockerignore` or any file
that reference `RAILS_MASTER_KEY` environment variable

You can pass environment variable to docker like:

```
docker run -e RAILS_MASTER_KEY=xxxxxxxxxxx -it myimage bash
```

...or link master key in docker-compose.yml :

```
---
version: '2'
services:
  rails-app:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - /tmp/master.key:/app/config/master.key # Given you build your project in `/app` in docker file
```

....or env variable in docker-compose.yml:

```
---
version: '2'
services:
  rails-app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      RAILS_MASTER_KEY: 'xxxxxxxxxxxxxxxxxxx'
```

#### CI & Servers

* Never log value of `RAILS_MASTER_KEY` anywhere (e.g. Jenkins logs, CI logs)

#### General concern

Yes Rails credentials are encrypted, that doesn't mean that file is non breakable if the file gets to the the wrong hands.
It's ok to store some development or test configuration there. But never store anything that may do harm
on production (e.g. production postgres database password)

> use Enviroment variables on production server for critical passwords,
> API keys, ...

Think!

### Regenerate key

Was your master key compromised? You want to generate new master.key?

Currently there is no "edit password" feature, you need copy original
content of the credentials, remove the enc
files and regenerate fresh credentials file ([source](https://github.com/rails/rails/issues/32718))

* step 1 copy content of original credentials `rails credentials:show`
* step 2  move your `config/credentials.yml.enc` and
`config/manter.key` away (`mv config/credentials.yml.enc ./tmp/ && mv config/master.key ./tmp/`)
* step 3 run `EDITOR=vim rails credentials:edit`
* step 4 paste copied values from original credentials
* step 5 save and commit `config/credentials.yml.enc`

> note! `EDITOR=vim rails credentials:edit` may not work if you require
> credential value in some file (e.g. in config/database.yml`)


### How secure are Rails credentials?

Point of Rails credentials is to help developers be more productive by
`git commit` all credentials (both development and production). Once
something is pushed on `git` it's there. Anyone with the copy of the
repo has the encrypted file.

So is it secure to store production credentials in Rails credentials ?

Rails uses `AES 128-bit` for credentials and in theory it takes several decades to crack this encryption.

There are many opinions on whether AES can be cracked. Short answer
"It can". Question is how long will it take with what technology in what era (10
years ago technology ? Today technology ? With technology in next 10 years ? )

**My opinion**:

Yes it's safe but it's like parking an expensive
car on rails of abandoned train track. Yes the train should not go there but
you will feel uncomfortable the entire time.

If it's a private project with couple of hundred users no one will spend resources on server farm
to crack your credentials. And by the time the project will grow big you
will probably have different db credentials => old file is no longer
valid.

If you are building a "Bank" application that will run for couple of
decades with same DB passwords, maybe that's not the best way
how to store DB password. It's still ok to store some small non core
credentials (E.g Sendgrid token).

In overall most security breaches happens because people are stupid.
Maybe your colleague has non encrypted laptop and after finishing his
employment with your company he will not erase the project as he was
asked to. Down the line 5 years later he throw away his non-encrypted
laptop with the entire project with safely encrypted Rails credentials
but also with the `config/master.key` still on same drive. Laptop &
drive will get to scrape yard somewhere in 3rd world countries where there are
organized gangs targeting such forgotten hard disks for information.

My point is: Think before you store something in Rails credentials. How
will the project evolve ? What personalities of developers will have
access to `master.key`? What are your company security policies?

> To be paranoid is on a job description of a senior web-developer.

### Discussion

* <https://www.reddit.com/r/ruby/comments/9211gs/rails_52_credentials_cheat_cheat/>
* <http://www.rubyflow.com/p/8f86uq-rails-credentials-cheat-cheat>

