---
layout: posts
title:  "Example: Using after_reply with Puma"
date:   2023-07-29 08:00:00 -0400
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

  "Hello Earthlings"
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

## Testing

How did we ensure that our `after_reply` logic was tested?

First, we ensured that our `Processor` class was fully testable independent of any request framework or middleware implementation. It sounds obvious, but to this point our observability gem had grown organically, and specs testing underlying logic were intertwined with specs for the middleware implementation. Refactoring the gem and the specs put us in a place where we could test the two separately. 

When `Processor` is used directly within the middleware, testing on middleware is easy as we can `instance_double` or `spy` the `Processor` and expect to have received `:call`. Moving to after reply means `call` will no longer be called in a context where we can directly observe, instead taking on faith that Puma is invoking `call`.

What we _can_ do is ensure that the middleware successfully adds `Processor.new` to the `env["rack.after_reply"]` array. This currently requires a bit of monkey-patching, as none of the current Rack mocks (used in middleware specs) allow access to `env` _after_ the request completes.

Here's how we monkey-patched it:

```rb
module Rack
  class MockRequest
    def request(method = GET, uri = "", opts = {})
      env = self.class.env_for(uri, opts.merge(method: method))

      if opts[:lint]
        app = Rack::Lint.new(@app)
      else
        app = @app
      end

      errors = env[RACK_ERRORS]
      status, headers, body = app.call(env)

      # Here's where we monkey-patch - MockRequest typically returns a MockResponse
      # In this case, we don't need a full Response object, we just need to be able
      # to inspect `env`
      # MockResponse.new(status, headers, body, errors)

      # Instead we return an array that includes `env`
      [status, headers, body, errors, env]
    ensure
      body.close if body.respond_to?(:close)
    end
  end
end
```

Then we setup our middleware test using Rack, invoke a request, then inspect `env` to make sure
it includes the expected `Processor` instances.

## Caveats

A few things to think about when deciding if `after_reply` is right for your use case:

1. Completion is _not_ required for a successful response to be returned: the response will be returned regardless of whether or not your `after_reply` logic completes successfully 
2. Your application can handle a moment of latency in a thread _before picking up the next request_: the Puma worker will still need time to complete `after_reply`, the apps in our case are sufficiently scaled so introducing `after_reply` had little effect on our utilization, but if you're working on a tighter scaling budget it's good to keep an eye on those metrics
3. Explore whether or not an async job (like Sidekiq) is a better fit; in our case of gathering observability metrics, tag collection required the most time, and enqueuing a job would have required running tag collection _before_ enqueueing, thus negating all the performance benefits. If you need to guarantee that the logic executes (or can be retried), and can ensure that the actual time-intensive tasks can be deferred, then async jobs are the better choice. 

## Wrapping up and results

`after_reply` is a great way to improve a bit on performance _if_ the time-intensive parts of your code can happen after a request completes. 

I mentioned using `rack.after_reply` in our observability tooling. It was satisfactorily performant operating as middleware, until we made one change which would increase the volume of metrics collected. There was no noticeable impact in many of our services. But in one of our core services it resulted in a 20 millisecond latency increase across p90 and p95. It also noticeably propagated upstream to one of it's heaviest callers (a consumer-facing app). 

After moving to `after_reply`, we were able to reintroduce our initial change with no increase in latency. In fact, moving to `after_reply` bought us a 2-3 millisecond improvement on our p90 and p95 latencies!
