---
layout: post
title: Using Timeouts in Ruby
---

Here's a fun one: using Ruby's built-in `Timeout` support. Whenever you're
in a production environment where you may be using long-running network
requests (or any expensive operations), be sure to add a timeout.

Heroku only gives you 30 seconds to complete your request, and if you exceed
that window, you'll get a SIGTERM. Instead, `Timeout::timeout` will raise an
exception that you can handle with your favorite
[error notifier](https://raygun.io/).

Here it is in action:

```ruby
require 'timeout'

Timeout::timeout(5) do
  sleep(10)
end

#=>Timeout::Error: execution expired
#=>from (pry):9:in `sleep'
```
