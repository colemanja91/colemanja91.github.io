---
layout: posts
title:  "Dumping Data as JSON and Field Deprecation in Active Record"
date:   2022-06-25 07:30:00 -0400
categories: rails
---
Our team recently came across a fun little quirk in the `as_json` method of an ActiveRecord model.

We're in the process of deprecating a set of fields, and had set up the following:

```rb
class OurThing < ApplicationRecord
  belongs_to :our_other_thing
  ...
  def our_old_field
    raise "`our_old_field` is now deprecated"
  end

  def our_old_field=(newval)
    raise "`our_old_field` is now deprecated"
  end

  def as_json(options = {})
    options[:except] << :our_old_field
    super(options)
  end
  ...
end
```

This took care of almost everything, as this field wasn't intentionally being used anywhere. The `as_json` method extension gives a nice way to ensure that this field is never returned in that payload.

This fell apart when including `OurThing` as part of another model:

```rb
our_other_thing.as_json(include: :our_thing) 
```

This bypasses the exceptions we built in to `OurThing#as_json` and attempts to return `our_old_field` as part of the payload, which in turn raises our deprecation error (thankfully we caught this in our specs).

The solution is to explicitly-exclude `our_old_field` in these cases:

```rb
our_other_thing.as_json(include: { our_thing: { except: :our_old_field } })
```

Might be a bit onorous to do this everywhere, but hopefully cases like this (which are essentially a table dump) are rare - it certainly drew our attention to places in our code where we need to be more intentional.
