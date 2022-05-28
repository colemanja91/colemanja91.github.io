---
layout: posts
title:  "Capturing Redirect Metrics in Rails"
date:   2022-05-28 07:59:44 -0400
categories: rails
---
When you're cleaning up a monolith Rails app, it's essential to have usage metrics to know what pieces of code are safe
to remove. Tools like Datadog APM and New Relic provide great insight on controller utilization, and [Coverband](https://coverband.io/)
shows hit rate on other areas of your code, like services and helpers. 

But when you're dealing with route cleanup, there's one piece that's pretty elusive - redirects. Well-used monoliths tend
to end up with large sections like this in `config/routes.rb`:

```rb
get "/old-home" => redirect("/home")
get "/other-old-home" => redirect("/home")
get "/seo-friendly-link" => redirect("/product-page")
...
```

It's difficult to get usage stats on these redirects because the logic is set when Rails is initialized, to keep
computational overhead low. And when you're actively trying to reduce cruft (or reduce the footprint
of an app for decommissioning), it's hard to have confidence that you won't break an important piece of business functionality.

One option is to setup a controller specifically for your redirects:

```rb
# app/controllers/redirects_controller.rb
class RedirectsController < ApplicationController
  def home
    redirect_to "/home"
  end

  def product_page
    redirect_to "/product-page"
  end
end

# config/routes.rb
...
get "/old-home" => "redirects#home"
get "/other-old-home" => "redirects#home"
get "/seo-friendly-link" => "redirects#product_page"
...
```

Then all redirects will be captured by default by APM. But this feels a bit heavyweight for something as simple as a redirect. 

Another option is passing a block into `redirect`:

```rb
get "/old-home" => redirect {|_, request|
  $stats.count("redirects", tags: {path: request.path})
  "/home"
}
```

This works because the block isn't evaluated until runtime, which is when we need to capture stats.
But it's tiresome and cumbersome when you have a _lot_ of redirects to track.

Instead, we add this bit of magic to the top of our `config/routes.rb`:

```rb
def redirect(*args, &block)
  super(*args, {|_, request|
    $stats.count("redirects", tags: {path: request.path})
    yield
  })
end
```

On a related note, if you have more complex logic to run at redirect, there's something nice tucked away in the 
[Rails redirection logic](https://github.com/rails/rails/blob/01f58d62c2f31f42d0184e0add2b6aa710513695/actionpack/lib/action_dispatch/routing/redirection.rb#L184):

```
# Finally, an object which responds to call can be supplied to redirect, allowing you to reuse
# common redirect routes. The call method must accept two arguments, params and request, and return
# a string.
#
#   get 'accounts/:name' => redirect(SubdomainRedirector.new('api'))
#
```

Which lets us do something like this:

```rb
# app/services/redirect_service.rb
class RedirectService
  def initialize(path)
    @path = path
  end

  def call(params, request)
    # some bit of complex logic
    @path
  end
end

# config/routes.rb
get "/old-home" => redirect(RedirectService.new("/home"))
```

At that point, though, you might consider just making another controller.
