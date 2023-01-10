---
layout: posts
title:  "Unhappy Paths Matter"
date:   2023-01-09 18:30:00 -0400
categories: product
---

Most product, design, and engineering folk are well-aware of their app's **Happy Paths** - that is, under all the right circumstances, the imagined optimal set of actions that your users take when interacting with your service (and the subsequent benefits they gain from those actions). The Happy Path is a critical part of any app, and most teams recognize this and take time to optimize it.

After several years of thought, I believe that the majority of issues that apps and services encounter when trying to increase adoption lie not in the Happy Paths, but rather in the **Unhappy Paths**. (In some cases, I've also heard them called _broken windows_).

An Unhappy Path is any hurdle, bug, or poor UX that occurs when a user attempts to take an action which hinders them from going down the Happy Path.

Unhappy Paths are different from normal UX friction. An example of normal friction is during, say, an onboarding flow when requiring the user to enter payment information. This step typically has a large drop-off - when people see the credit card field, they change their mind and decide it's not something they want to spend money on. 

By contrast, an example of an Unhappy Path might happen if the user enters their payment information, but your payments processor's API returns an error, and you don't give the user any indication of what went wrong or how to fix it. In this case, your product was compelling enough to get them over one of the biggest drop-off points, but then something happened that prevented them from moving forward.

## Not the Priority

Unhappy Paths are usually not the most glamorous investments in a product. Changing the layout and design of a page, overhauling an onboarding flow, or integrating a new SSO provider are usually more exciting options than improving error messages. As a result, Unhappy Paths are often overlooked during design efforts, prioritization, and working towards launches. This trend carries into ongoing optimization; "We really want to see if New Feature X works, so let's prioritize that. We'll get to fixing this error message later."

Also, when attempting "relaunches" or "V2.0" iterations, lessons learned around Unhappy Paths tend to be the first things forgotten. This is [one of the biggest reasons that Netscape Navigator lost in the browser wars](https://www.back2code.me/2016/11/netscape-the-rewrite-big-mistake/).

As always, context matters - if you're working on your MVP, then tracking down all the possible Unhappy Paths is probably not the best investment to make. After you've proven the concept, though, it can sometimes have a huge impact.

## Lack of Analytics

Both product and engineering metrics have a tendancy towards ignoring Unhappy Paths. 

Let's look at a basic conversion rate for a sign-up page. The two most common events to see would be "Page Visit" and "Signed Up." The conversion rate would then be `count("Signed Up") / count("Page Visit")`. 

For this example, let's say our conversion rate is currently 50%. A typical focus would be attempting to increase the conversion rate: adding SSO proviers, reducing the amount of info required, changing verbiage or designs. 

What we're missing in this context is info about the Unhappy Paths. How many visitors to the sign-up page _attempted_ to sign-up? This info has a tendency to be untracked, and could be very insightful. If the attempted sign-up rate is significantly higher than the conversion rate, that is an indicator of unoptimized Unhappy Paths.

This doesn't just happen with product analytics events, though. It happens with engineering monitoring as well. 

Engineers tend to do a great job of handling error cases - we make sure that exceptions are caught, and raise the appropriate HTTP error status code. After that, we have a tendency to be hands-off. For example, a backend service may have a high rate of 4XX error codes against a particular endpoint, but in theory, only valid / actionable requests should be coming through. This is another smell of Unhappy Paths. 

Remember, your system gracefully handling an error case doesn't necessarily mean that the user will understand why it didn't work.

## Engineering <-> Product Friction

That said, engineers also tend to be the closest at ground-level to the Unhappy Paths. We build the data models and APIs and want them to be robust, so we research and anticipate all the ways they might fail. 

This has a tendency to put us at-odds with Product folks. Engineers want to build in time for development and proper handling of error cases; product managers think this is making the project too complex and adding too much build time. (I say this having been both an engineer and a product manager).

Like I mentioned earlier, context matters - sometimes it is better to get something out fast, and refine it later. The problem is, most veteran engineers know that "refine it later" time will never come. There will always be something more exciting to be prioritized; they've experienced this over and over, and are too cynical to believe that there will ever be time to "dig into that later."

## Closing Advice

* **Document error cases in your APIs**
  + At the very least, this will help front-end engineers understand the types of possible errors, and they can attempt to cushion the UX
  + [Facebook's API docs do a great job at this](https://developers.facebook.com/docs/graph-api/guides/error-handling/)
* **Build observability for all outcomes**
  + Ensure you can properly dig in to all possible results from your API endpoint
    - What percentage of requests have a success vs. non-success result?
    - Out of the non-successes, what are the most common errors?
    - If a single HTTP code possibly represents multiple error cases, can you easily see which specific errors are occurring (as opposed to just the status code)?
  + Ensure you can view attempt vs. success rates in your product analytics
    - Out of all the users that experience errors, can you tell how many of them eventually succeed? Or how many abandon your product completely?
  + If supporting multiple platforms (i.e., web, Android), can easily view your observability either in aggregate or drilled down to a specific platform?
  + Most importantly - once you can answer "yes" to these, revisit them regularly to ensure that they are informing your prioritization
* **Be explicit about error cases in your API docs**
  + **This one is for the engineers**
  + We _all_ have a tendency to half-ass our API docs; this makes it harder to identify the Unhappy Paths
  + Ensure that, where appropriate, consumers of your API (i.e., front-end devs) can
    - Identify what went wrong
    - If possible, instruct users on what needs to be changed/fixed
* **Define a follow-up period for new launches to find and address Unhappy Paths**
  + **This one is for the PMs**
  + If you want your engineers to trust the process of moving quickly, you have to ensure dedicated time after launch to identify and address Unhappy Paths
    - This must be consistent over time; this is a matter of engineering and product trusting each other, and the end result is only better for your users

_[A previous post, in case you want to red more ranting about observability (and dinosaurs!)](https://colemanja91.github.io/observability/dinosaurs/)_