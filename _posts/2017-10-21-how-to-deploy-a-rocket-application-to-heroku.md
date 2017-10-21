---
title: How to Deploy a Rocket Application to Heroku
---

[Rocket](https://rocket.rs/) is a web application framework for the
[Rust](https://www.rust-lang.org/) programming language.
[Heroku](https://www.heroku.com/) is a platform-as-a-service provider that makes hosting
web applications easy and, at the (low-traffic) hobby level, free.
Deployment is as easy as pushing to a special git remote repository.

I recently deployed [my first Rocket application](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust) to Heroku.
Here's how.

Since I hadn't used Heroku in a long while, I first needed to install the `heroku`
Command Line Interface (CLI) and log in.

```
sudo apt-get install heroku
heroku login
```

If you aren't using Linux and apt, see [Heroku's instructions](https://devcenter.heroku.com/articles/getting-started-with-ruby#set-up) for installing the Heroku CLI.
If you don't already have an account, you'll need to register on their web site first.

From the root directory of my already written application, which is the root of its git repo,
I created the Heroku application, which I named `todo-backend-rocket-rust`,
using [this Rust buildpack](https://github.com/emk/heroku-buildpack-rust).

```
heroku create todo-backend-rocket-rust --buildpack https://github.com/emk/heroku-buildpack-rust.git
```

That command automatically added a git remote called `heroku`.

Next I [added two new files](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/598e4dd716fa64c7ebde14a39d7049c2f7b07856), the Procfile and a RustConfig.
By default, the Rust buildpack will use the latest stable Rust,
but Rocket requires nightly Rust.
I specified `VERSION=nightly` in RustConfig to override the buildpack's default.
The Procfile tells Heroku how to run your application.

I added one unconventional thing to my Procfile to get Rocket to run on
the Heroku-designated port. Rocket will look for a `ROCKET_PORT` environment variable
to override its default or `Rocket.toml`-configured port.
Since (AFAIK) the Heroku port is neither configurable nor predictable nor consistent,
I couldn't configure it in `Rocket.toml` or rely on Rocket's default,
but I didn't want to have to change the code to look for a different environment variable,
if I could avoid it.
I tried assigning `ROCKET_PORT=$PORT` in the Procfile, and it worked.

```
web: ROCKET_PORT=$PORT target/release/todo-backend-rocket-rust
```

Since the Heroku application name I chose was available,
I knew the application would be available at
`https://todo-backend-rocket-rust.herokuapp.com/` once I deployed it,
and my application needs to be configured with its public base-URL,
so I [committed the necessary config](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/640a437f07f3894608541ad075ba2963258abb95)
to `Rocket.toml`.
(Note that `base_url` is not standard Rocket configuration;
it's something custom that my application [uses](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/blob/640a437f07f3894608541ad075ba2963258abb95/src/main.rs#L18).)

Then I set the `ROCKET_ENV` heroku configuration variable for my application
to tell Rocket to load the right configuration from `Rocket.toml`.
Heroku configuration values are exposed to applications as environment variables.

```
heroku config:set ROCKET_ENV=production
```

Then I was ready to deploy...

```
git push heroku master
```

and [see my application up and running](https://www.todobackend.com/specs/index.html?https://todo-backend-rocket-rust.herokuapp.com/)!
