---
layout: posts
title:  "Dinosaurs and Observability"
date:   2022-06-19 12:00:00 -0400
categories: observability
---
_Jurassic World Dominion_ just came in movie theaters. It is absolutely terrible, and I love it. 

In a related note, the new book _[Observability Engineering](https://www.amazon.com/Observability-Engineering-Charity-Majors-ebook/dp/B09ZQ6FHTT)_ just came out. It's great, and so far I've loved reading it.

How are they related? Easy. The _Jurassic Park_ franchise has one of the best examples of monitoring and observability ever. It's not in the movies though - this story comes from the original sci-fi horror novel by Michael Crichton. (This was one of the first full-length novels I ever read, so it holds a special place for me).

In both the novel and the original 1993 movie, one plot point revolves around the characters learning that the genetically-engineered dinosaurs have been breeding on the island. This is a shock because the creators of the dinosaurs were 100% certain that they couldn't breed, thus giving the scientists a false sense of control. 

In the movie, this entire plot point is addressed in one short scene where the characters find hatched dinosaur eggs with little dinosaur footprints leading away from the nest. In the book, it plays out a bit differently. 

There's almost a full chapter describing the computer systems used in the park (it's a big part of the plot, and general audiences weren't as familiar with computers in the late eighties). One of these systems is a video-based recognition system that counts dinosaurs, and raises alerts if the count of dinosaurs on the island is different than what's expected. (This sort of technology was pure fiction at the time of the novel, but with modern ML is [very realistic](https://www.youtube.com/watch?v=8bMX6rtw6qg)). 

When the novel characters find evidence of dinosaur eggs, this sparks a debate. On one hand, there's hard, physical proof of dinosaurs breeding. On the other hand, there's so much confidence that the park's computer system would have alerted them if there were more dinosaurs than expected.

One character, Dr. Ian Malcom (played in the movies by the one-and-only Jeff Goldblum), sees the issue. He asks one of the park operators to increase the expected count of total dinosaurs by one. They do that, run the counting program, and it completes without any errors - showing that there is at least one more dinosaur than expected on the island. They keep increasing the expected count, and eventually find there are dozens more dinosaurs than expected (including at least 4x the expected number of velociraptors)! 

There are three things here that Dr. Malcom recognized:

* The intended business value of the monitoring - each dinosaur was an investment worth hundreds of millions of dollars. If there were any issues, such as a dinosaur dying, the park operators needed to know right away. 
* The computational cost of running this piece of software was extremely high (this was way before the cloud, so everything was on-prem). That meant it had to be optimized in every way possible. 
* Everyone running the park was starting with their assumption that the dinosaurs couldn't breed. 

This led to a simple optimization of the program - it would count dinosaurs until it reached the expected number, then would spit out a success message. To let the program continue it's attempt to count dinosaurs would have been an unnecessary spend, because, in their eyes, there would never be more dinosaurs than this particular number, and it acheived their goal of letting them know if any dinosaur had died.

This really is a perfect example of how a lot of monitoring is implemented: 
* There's a specific business question that needs to be answered
* That question has a lot more complexity than it may seem on the surface, so simplifying assumptions are made
* The resulting answer narrows the field of vision, blinding everyone to other questions that may need to be asked
* The resulting answer is also used as proof of the original assumptions (this is probably why Dr. Malcom - the mathematician character - was the one to spot this issue)

At Jurassic Park, they had monitoring - they could watch for a particular case they knew and cared about. But they did not have observability - they couldn't answer questions that they couldn't think of. And it quite literally came back to bite them.
