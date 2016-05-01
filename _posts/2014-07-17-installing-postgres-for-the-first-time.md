---
layout: post
title: Installing Postgres
---

These are the steps that I typically follow on a Mac to install Postgres from scratch:

```
brew install postgres
```

> Don't have [brew](http://brew.sh/) (or know what it is)? Fear not: Brew is a package manager (a nerdy App Store) that can be installed with the following command:

> `ruby -e "$(curl -fsSL https://raw.github.com/Homebrew/homebrew/go/install)"`

Once Postgres has been installed, you'll need to start the server (which the output of brew should tell you how to do). Decide whether you want it to start every time that your computer boots, otherwise you can use a more manual process and start the service on demand.

The next step is to create a user that your web applications can use to connect to the database. Typically when working with rails applications, the convention seems to be creating a user named `rails`. You can do so with the following command:

```
createuser rails
```

> An important note: this isn't actually creating a user on your machine, it's simply making a user inside of Postgres. If you ask me, this should be called something like `createpostgresuser`. If you're more interested in what `createuser` can do, you can find out more with `man createuser`.

Next up, give that user some superpowers. You'll need to make your new user a superuser in order to have the ability to create new databases with commands like `rake db:create` or `rake db:migrate`. To do so, open up a Postgres console to the default table:

```
psql template1
```

> `template1` is the default table for all fresh Postgres installations.

```sql
ALTER USER rails WITH superuser;
```

### BAM!

You should be done. You can quit out of `psql` with `\q` or `CTRL` + `d`. From here, commands like `rake db:create` and `rake db:migrate` should work. If they aren't, make sure that your `database.yml` file is using the `rails` user. A sample configuration for Rails 4 can be seen here:

```yaml
development:
  adapter: postgresql
  encoding: unicode
  pool: 5
  database: testapp_development
  username: rails
```

> Note that Rails 4 actually uses the `<<: *default` directive so that you can share similar configurations between environments. In the above example, I've removed that to show what the actual parameters look like for the development environment.
