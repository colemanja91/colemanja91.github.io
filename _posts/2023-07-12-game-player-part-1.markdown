---
layout: posts
title:  "Building a Game Player with ChatGPT and Rails - Part 1"
date:   2023-07-12 12:30:00 -0400
categories: rails
---

Watching Twitch streamers and their highlight videos is a guilty pleasure of mine. [DougDoug](https://twitch.tv/dougdoug) does a lot of streams where the main idea utilizes AI, typically to help play a video game. One of his recent streams involved using [ChatGPT to beat a children's point-and-click adventure game](https://www.youtube.com/watch?v=W3id8E34cRQ) (warning, this is a long video, but very entertaining!)

DougDoug scripted together multiple pieces with Python to make this work:

* A script that maintains the initial prompt, historical context, and sends the latest prompt to ChatGPT for a response
* Speech-to-text generator to interpret DougDoug speaking into a mic to provide context for ChatGPT's next prompt
* Text-to-speech generator that reads out ChatGPT's response in a character voice

The whole thing was written as a very impressive Python script, but that left it prone to crashes and a few other issues. As I was watching, I couldn't help but think through how to architect this into something more resilient and flexible.

_Note_: this is not a criticism of DougDoug's programming, many of the things I note here as "issues" led to the most entertaining parts of his stream. This is simply me exercising my problem exploration and resiliency muscles to keep them sharp, and would generally result in a less funny outcome :)

# Basic Architecture / Data Flow

Let's start by looking at the basic pieces of information that move around:

1. _Prompt Context_: What are the rules ChatGPT must follow during this conversation?
2. _Input Audio_: Audio file from the user where they provide ChatGPT with context of the next decision it needs to make
3. _Input Text_: A text-generated version of _Input Audio_
4. _Response Text_: ChatGPT's output
5. _Input Prompt_: A combination of _Prompt Context_ (always included), historical _Input Text_ and _Response_Text_ (included to the point of reaching ChatGPT's prompt size limit), and current _Input Text_ (always included)
6. _Response Audio_: Audio file generated from _Response Text_

The _Input Prompt_ information is the most complex; ChatGPT has to be provided with conversation history inside of a request to be able to reference that history (at least, when using the API). 

To look at it visually over the course of a few requests (borrowing from how DougDoug illustrated the process):

# How We'll Build It

A few considerations I have:

* Store conversation history so we can reconstruct the correct _Input Prompt_ regardless of server crashes
* Retain the option to "wipe" memory and start from scratch
* Ideally, provide some kind of UI

Stretch goals:

* Allow the user to toggle whether both _Input Text_ and _Response Text_ are included in _Input Prompt_, or only one of them is
* Ability to track multiple "characters" separately (imagine a D&D campaign!)
* Show how much history was included with each request (i.e., exactly what was sent to ChatGPT?)
* "Pogress" bar showing which step is currently working

The speech/text translation pieces arguably make this project better suited for Python, but the remaining pieces are straightforward in Ruby on Rails. That's where I'm most comfortable, so we'll be building inside that framework. For resiliency, we should put some components in background jobs (basically, anywhere we call a 3rd party service).

# Data Model

Since we want to eventually track multiple characters, we should start with that as our top-level model, as everything else will branch out from that.

_Prompt Context_ is a single large text field associated with a character; we can store this in the character model.

_Input Text_ and _Response Text_ are both text values. They should be stored relationally to a character. One question: should they have their own individual tables, or be stored in one table with a column to distinguish the types? Well, to construct _Input Prompt_, we'll need to be able to sort in chronological order regardless of type. We also need to filter to one type or the other to support our stretch goal. This is easier to accomplish if we store them in a single table.

_Input Audio_ and _Response Audio_ will only be temporary, so we can store them to the filesystem in a temp folder. If we were planning to run this as an external-facing web app, we'd probably want to store it on Amazon S3 or some similar service.

_Input Prompt_ is the fun one because we have two viable options. We could always generate the value dynamically and let it be ephemeral, or we could store it after generation. There is no functional difference in terms of our interactions with ChatGPT, but if we ever change the generation rules then we'd lose visibility into what we had previously sent over. Just for the sake of debugging, we'll store _Input Prompt_ so we can reference it in the future, but we won't prioritize visibility. 

Given that, we might consider keeping _Input Text_, _Response Text_, and _Input Prompt_ in some sort of general "text events" table. Is this a good idea? Well, if we ever wanted to display a full history of these fields in chronological order, that might be helpful. But there may be additional context related to _Input Prompt_ we might want to store, such as ChatGPT API version. So if we do store _Input Prompt_, we'd likely want to do that in a separate table.

# Order of Operations

Now that we've thought through these details, where do we get started? 

_Input Audio_ and _Response Audio_ are very much add-on pieces to the rest of the app, so we can save those for later. 

The character model - along with _Prompt Context_ - should probably come first, given that everything else in the data model comes from those. 

Next we can model out _Input Text_ and _Response Text_, then we should be good to setup _Input Prompt_ logic and construct our ChatGPT calls.

Once the modeling is done we can create our APIs. I didn't dig into the choice for this, but I'll use GraphQL (partly because I want to experiment with using GraphQL Streaming). With the API in place, we can then craft a basic UI where we can:

* Create a new character
* Define and edit our _Prompt Context_
* Type in an _Input Text_
* Get back and display a _Response Text_

# Next Steps

As I build this out I'm planning to write posts here as well as sharing PRs used in the process! 
