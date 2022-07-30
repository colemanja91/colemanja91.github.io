---
layout: posts
title:  "Debugging Network Issues with AMC Theaters"
date:   2022-07-30 11:00:00 -0400
categories: storytime
---

I've been on vacation this week, which means two things:

* I've been consuming more media (TV, movies, video games) than normal
* I've been looking for programming-related things to think about (I can't really turn off that part of my brain)

Yesterday, these two just happened to overlap! 

I bought a ticket to see _DC League of Super Pets_ at the local AMC. I'm a pretty frequent customer there, given that it's only a couple of minutes away from home, and I always use the AMC Theaters app.

In this case, when I got to the theater, I opened the app while sitting in my car so I could pull up my ticket, but got the error below:

![Screenshot of the AMC Theaters app displaying the text "Error connecting to AMC Theaters. Please check your internet connection and retry."](../../../assets/images/amc_app_no_connection.png)

OK, I did the normal troubleshooting steps:
* Made sure I had an internet connection (could access other apps with no problem)
* Turned my mobile network connection off then back on
* Cleared local storage/cache for the app
* Reinstalled the app
* Restarted my phone

Most telling, though, was that even attempting to sign in to AMC thru their website (on Android Chrome) wasn't working - there was no error displayed, it would simply show the loading animation then stay on the sign-in screen. 

Nothing worked, so I was thinking it was just a server issue with AMC. Then I remembered that they also email a receipt with the QR code and confirmation number; the confirmation number was enough to get me in, but strangely the QR code wasn't loading in the email itself. Also strange was the fact that, once I got in the theater, no one else seemed to be having any issues. When I got home, it seemed whatever the issue was had been resolved as the app could open just fine.

This morning, I was looking in to getting tickets for _Nope_ while at the local [coffee shop](https://fountcoffee.com/), and again the app was working fine, but on the way home I stopped off for a donut, and curiously, when I checked the app again, I got the same network connection issue. Hmmm. 

Things were starting to smell like an odd network issue rather than a server issue. When I got home, I ran an experiment - I checked the app while connected to my wifi and had no issues. Then I switched over to mobile data and boom! The issue was back. Remembering that there had also been issues with the web version, I connected my laptop to mobile data via hotspot, and was able to inspect the network calls: 

![Firefox dev tools showing multiple errored calls](../../../assets/images/amc_app_cors_graph.jpg)

Looks like pretty standard requests - GraphQL for API requests, Cloudinary for caching, and Google Analytics for tracking (Dear AMC: nice stack!). But they're all getting hit with what appears to be CSRF failures, or at least that's how the browser is interpreting them. We've also had to troubleshoot some issues around GraphQL and edge caching at Stitch Fix, so it makes sense to me that there could be some unaccounted-for edge cases. I also went back and double-checked the email receipt; the QR code is displayed via a GraphQL call, so that fits with everything else.

Two side notes on the email:
* I use [Spark](https://play.google.com/store/apps/details?id=com.readdle.spark&hl=en_US&gl=US) for my email, but it looks like using Gmail directly would result in Google caching the image, meaning I could still load the QR code if I opened the email in the Gmail app
* As a further compliment to AMC's stack - nice use of Salesforce Marketing Cloud for emails! 

Now, why would my particular mobile provider be having issues? I don't know exactly why, but it seems to be linked to my using Google Fi. When checking details on my IP, it appears that Google Fi, by default, masks some elements behind a VPN, so location-wise it looks like I'm in Mountain View, CA:

![Current IP v6 address showing in Mountain View, CA](../../../assets/images/google_fi_ip_no_vpn.jpg)

(As a note, I'm currently located in Cary/Raleigh, NC).

I don't typically have the Fi VPN enabled, so this was a bit surprising. Interestingly, I found that enabling the Fi VPN while on mobile data actually enables the AMC app and website to work correctly! I also checked out a few other websites that use Cloudinary for edge caching, and didn't see any issues, so I'm thinking it must have something to do with how Cloudinary is specifically configured for AMC (and very likely, specifically configured against their GraphQL implementation).

Also, the last time I saw a movie at AMC, I did not run into these issues, meaning the root cause can likely be traced back to some change made between 2022-07-16 (when I saw _Paws of Fury: The Legend of Hank_) and 2022-07-29.

So, in the spirit of the following [XKCD comic](https://xkcd.com/806/), I'm leaving these details here in case the engineers at AMC find them helpful. Definitely not urgent for me, as I've discovered a couple of workarounds, but might be good to look into in case it affects other customers. 

![I recently had someone ask me to go get a computer and turn it on so I could restart it. He refused to move further in the script until I said I had done that.](https://imgs.xkcd.com/comics/tech_support.png)
