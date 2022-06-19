---
layout: posts
title:  "How I Learned to Stop Worrying and Love Testing in Production"
date:   2022-06-18 09:00:00 -0400
categories: engineering
---
Confession: I've done a lot of live testing in production during my career.

Some of the first apps I built were in Oracle Application Express (with a hodgepodge of PL/SQL stored procs,
Python scripts running from cron jobs, and fumbled-together jQuery). I only had a single production environment,
so changes were made live while people were using the apps. It always felt precarious, and every change was dangerous. 

Career progression ended up with better test environments - at one point, I built a system for turnkey AWS environments,
where engineers could quickly spin-up a QA server specific to the branch they were working on, and which would
be deleted as soon as that branch was merged or deleted.

Now, as part of an ecommerce company with hundreds of thousands of users, I have a new love: testing in production. 

_all the experienced engineers out there can take a moment to breathe, or go get a drink_

Testing in prod is our officially-sanctioned method of QA and testing. It took me a minute to get used to it. 
Now I want to share what I've learned.

## Where Staging Falls Apart

What's wrong with deploying to staging and taking a quick look? Do we even want to get rid of cursory checks? 

Well, it depends on the complexity of your team (and by extension thru Conway's Law, your app).

With a simple team - a couple of developers - it's easy to coordinate use of staging. Work streams will rarely overlap,
and when they do, people usually know about it in advance. 

If you have a more complex environment, it goes wrong very quickly. With multiple teams working in a single monolith,
you may have to coordinate and re-test work if people happen to be working in similar areas of the codebase (or even if they don't - 
monoliths sometimes have surprising links between seemingly unrelated areas). With multiple teams working in a service-based architecture,
you end up worrying about whether or not a service you're calling is up-to-date with production or not. 

(Consistency between services is a related issue on it's own - and testing tools [like Pacts](https://multithreaded.stitchfix.com/blog/2015/11/23/consumer-driven-contracts/) an help).

In short, there's a lot of coordination work which can quickly go wrong, leading to either longer development times or to false confidence
in the testing process. 

## How to Do It Right

I don't recommend just blasting away your staging environments with no plan. The hidden power of testing in production is
that it forces investments in other far more valuable areas. 

### Confidence in Dev Processes

* **Code Reviews** - Yes, I know, Captain Obvious here. But it's a baseline requirement - we all need a second set of eyes.
Not because those eyes are better, but we all have potential to learn things ("have you thought about doing it this way")
or to just get tired ("won't this cause a cartesian join?").
* **Code Ownership** - You need engineers who understand a system and are thinking about it's long-term health. Even when
that system is actively being deprecated, you need intentionality behind that process, which can only come from people
who have decided to care about a particular codebase. 
* **Variations of TDD** - I don't always believe in full Test Driven Development (i.e., always writing the tests first,
writing one failing test at a time); sometimes, you just don't need that level of granularity. But thinking about tests
early-on is amazingly helpful. It's a great way to learn a new system. It's a great way to give others the ability to
create PRs in your systems - you can start by reviewing the tests, understand the use case (and that it's not breaking
anything else), then move on to the implementation. 

### Confidence in Dev Tooling

* **CI** - 
* **Inter-Service Testing** - You need confidence that when you make changes to your systems, you won't screw over others who depend on those systems. This is where [Pacts](https://multithreaded.stitchfix.com/blog/2015/11/23/consumer-driven-contracts/) can
be super-helpful (if occasionally annoying, but hey, that's another place to improve the tooling!)
* **CD** - As engineers, we often think of continuous deployment as a convenience for getting our new code to production. But it also
has a far more important purpose - quickly rolling back code to the previous good state. If you're testing in production, this is critical
to building confidence; if it takes 10 seconds to rollback a bad code change, you're more likely to try something, as opposed to if it takes
10 minutes (which could leave your app in an unusable state). 

### Confidence in Monitoring and Observability



### Verifying the Thing

* 

* Understanding how customers use the app
* Forces regular use from a customer-perspective

### Testing Through Volume

* Catch more issues faster

Think of it like a non-automated, sometimes-unintentional form of Chaos Monkey. 


## References
