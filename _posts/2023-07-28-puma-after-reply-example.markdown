---
layout: posts
title:  "Example: Using after_reply with Puma"
date:   2023-07-28 08:30:00 -0400
categories: rails
---

Puma has a pretty interesting feature called `after_reply` - if there's a potentially costly operation that's not on the critical path to responding to a consumer, you can delay that operation until _after_ the response is returned. I've been digging into this lately trying to optimize our observability tooling. [GitHub did something similar with Unicorn](https://github.blog/2022-04-11-performance-at-github-deferring-stats-with-rack-after_reply/) (I'm taking inspiration from them).

What's hard to find was an actual implementation example. After whipping up one, I figured I'd share.

## Installation

```
gem install puma
gem install sinatra
```

## Without after_reply

_filename: myapp.rb_

First we'll try something simple with inline execution:

```rb
require 'sinatra'
configure { set :server, :puma }

class Processor
  def call
    count = 0
    while count < 3
      puts "counting #{count}"
      sleep 1
      count += 1
    end
  end
end

get '/' do
  puts "log message"
  Processor.new.call

  "Hello wor"
end
```

Run it with `ruby myapp.rb` and hit the listener in a browser:

```
➜  ~ ruby myapp.rb
Ignoring debug-1.7.1 because its extensions are not built. Try: gem pristine debug --version 1.7.1
Ignoring rbs-2.8.2 because its extensions are not built. Try: gem pristine rbs --version 2.8.2
== Sinatra (v3.0.6) has taken the stage on 4567 for development with backup from Puma
Puma starting in single mode...
* Puma version: 6.3.0 (ruby 3.2.0-p0) ("Mugi No Toki Itaru")
*  Min threads: 0
*  Max threads: 5
*  Environment: development
*          PID: 31283
* Listening on http://127.0.0.1:4567
* Listening on http://[::1]:4567
Use Ctrl-C to stop
log message
counting 0
counting 1
counting 2
127.0.0.1 - - [01/Jul/2023:07:32:54 -0400] "GET / HTTP/1.1" 200 9 3.0250
```

Execution time was > 3 seconds - definitely noticeable in the browser. 

## With after_reply

Now let's switch it up to use `after_reply`:

```rb
require 'sinatra'
configure { set :server, :puma }

class Processor
  def call
    count = 0
    while count < 3
      puts "counting #{count}"
      sleep 1
      count += 1
    end
  end
end

get '/' do
  puts "log message"
  env["rack.after_reply"] = [Processor.new]

  "Hello wor"
end
```

Running and hitting it again:

```
log message
127.0.0.1 - - [01/Jul/2023:07:37:09 -0400] "GET / HTTP/1.1" 200 9 0.0075
counting 0
counting 1
counting 2
```

Response time is drastically improved, and we still get to run our non-critical-path code.

## Details

What do you actually need to implement `after_reply`? 

1. An object that implements `call`

The logic for `after_reply` invokes `call` on whatever objects you pass in, so the code you want to execute has to be triggered by that method.

2. Access to the request `env`

In Rails apps, you can access this in middleware:

```rb
class CustomMiddleware
    def initialize(app)
        @app = app
    end
    
    def call(env)
        env["rack.after_reply"] = [Processor.new]
        @app.call(env)
    end
end
```

or in a controller:

```rb
class MyController < ApplicationController
    def show
        request.env["rack.after_reply"] = [Processor.new]
        render json: {success: "true"}, status: 200
    end
end
```

Note that the value for `rack.after_reply` must be an array - to be safe, you should instantiate and add, in case you end up using `after_reply` in multiple places (this avoids accidentally overwriting any other usage of `after_reply`):

```rb
class MyController < ApplicationController
    def show
        request.env["rack.after_reply"] ||= []
        request.env["rack.after_reply"] << Processor.new
        render json: {success: "true"}, status: 200
    end
end
```

3. Pay attention to state

When instantiating `Processor` (or whatever your callable class is), be sure that the information you need is available after the request is returned.

For example, we use a handful of [ActiveSupport::CurrentAttributes](https://api.rubyonrails.org/classes/ActiveSupport/CurrentAttributes.html) instances to pass around customer- and request-specific information (i.e., making sure that certain data about the original request is always available to downstream services without having to explicitly pass as a parameter). But, we always explicitly clear these instances on completion of a request - when `after_reply` calls our `Processor`, those attributes aren't available anymore. 

To get around this, our `Processor` extracts the information it needs when we instantiate (`request.env["rack.after_reply"] << Processor.new`), but the heavy lifting still happens after the request is returned.

## Wrapping up

`after_reply` is a great way to improve a bit on performance _if_ the time-intensive parts of your code can happen after a request completes. 
