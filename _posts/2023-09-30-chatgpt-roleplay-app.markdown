---
layout: posts
title:  "Building a Game Player with ChatGPT and Rails - Part 2"
date:   2023-09-30 09:00:00 -0400
categories: rails
---

[DougDoug](https://twitch.tv/dougdoug) is a Twitch streamer (and programmer!) who recently did a fun stream making [ChatGPT play a children's point-and-click adventure game](https://www.youtube.com/watch?v=W3id8E34cRQ). It's a long but incredibly fun video, better than any movie currently in theaters. 

[I wrote a bit about the implementation and how I might approach it from a resiliency perspective](2023-07-12-game-player-part-1.markdown), but I didn't make it too far into actual build due to the whole damn staying-alive-and-having-an-emotionally-draining-job thing. But as stress relief for the last month or so, I have also been streaming on Twitch ([feel free to drop in and check it out](httpw://www.twitch.tv/allie_nord)), and I wanted to try something similar. 

This week I've been streaming development of my own implementation, which has been a ton of fun. Four evenings of development in total, and I streamed three of them, plus the inaugural day in which I used the new app to play _Hitman: World of Assassination_ - the VODs are linked below in case anyone wants to check them out:

* [Day 1 - Exploring the OpenAI APIs and doing initial Rails setup](https://www.youtube.com/watch?v=HbtbUY7WBJc)
* [Day 2 - Writing specs and building request/token-size min-max](https://www.youtube.com/watch?v=HfV8Mq9cLOY)
* [Day 4 - Writing GraphQL specs and finishing ReactJS front-end](https://www.youtube.com/watch?v=fLWhJhdagdA)
* [Day 5 - Testing the app by playing Hitman](https://youtu.be/Tl7ntREh188)

The source code for both backend and frontend are in these repos:

* [Ruby on Rails + GraphQL Backend](https://github.com/colemanja91/chatgpt-roleplay)
* [ReactJS Frontend](https://github.com/colemanja91/chatgpt-roleplay-ui)

## More Recent Lessons

It's worth noting that DougDoug recently did another "ChatGPT Plays" stream, where the implementation was much more robust. One side-effect is that the AI became less "crazy" over time - in the first stream, the message context would roll off (including the initial prompt with the rules for the AI), and ChatGPT would increasingly be "trained" on it's own responses. For the second stream, it appears that he built in a "system message" which would be present on all requests (OpenAI terminology - I referred to it as "Prompt Context" in my first post). This led to the AI responses being fairly consistent, and slightly less entertaining (DougDoug ended up doing some "brain surgery" during the stream to try to make it more entertaining).

I also finally spent some time looking at OpenAI's API docs (reading the manual - imagine that!) and reading more on LLMs, and learned a few important things:

* What I previously referred to as the "Prompt Context" (i.e., the initial message containing all the rules for ChatGPT) is now a supported feature called a "system message" - however, we have to explicitly include it at the start of every API request
* The "system message" _must_ be at the top of the message history in the API request - otherwise ChatGPT will interpret it as just another user input
* Usage/utilization is tracked via "tokens", which are an LLM way of breaking up language into processable chunks
* The max token limit for a ChatGPT 3.5 request is 4,096
* There are a few parameters that can tune the types of output ChatGPT generates - the main one of interest being [temperature](https://platform.openai.com/docs/api-reference/chat/create#temperature), which controls whether a response is more deterministic or more random (this is a float between 0 and 2)
  * Making ChatGPT API requests with a `temperature=1.8` or higher tends to crash the API; lower values are consistently faster

## Requirements

Thinking about an entertaining MVP for both a coding and streaming perspective, here are the minimum requirements I came up with:

* History for a particular "character" should be retained, so that I can come back later and pick up where I left off
* The "system message" should always be sent at the top of the `messages` array
* We should construct our requests so that we never exceed token limits
* Since I plan to use this while streaming, it's important for the input to via speech recognition (nobody wants to watch a streamer spend 30 seconds typing)

~For the MVP, I explicitly left out text-to-speech processing for the response - I can read it out myself for a while, and TTS API services are a tiny bit expensive.~ I subsequently decided, after I wrote this but 2 hours before I was going to stream Hitman, to add TTS as optional.

## Architecture

The data storage and OpenAI integration requirements are very easy to implement in a framework like Rails, so that's where I started (with Postgres for storage). 

Because I'm only planning to run this app locally, I'm not currently worried about Dockerizing, adding auth, monitoring, etc. 

The most extenuating requirement is around speech recognition. There are several providers that we could call from Rails, but they require us to already have an audio file to process. Ruby and Rails are very lacking in media-related tools (especially compared to Python). I briefly went down a dark path of trying to pipe commands to ffmpeg in the shell, but quickly gave up.

One option could have been to implement via Python, but I am not as familiar with those frameworks and it probably would have tripled implementation time (not something I wanted in this case due to it being a hobby project).

After some research, I settled on building a basic UI in React JS. James Brill has built a fantastic [React library that integrates with browser speech recognition tools](https://github.com/JamesBrill/react-speech-recognition), meaning that if I run the UI in Chrome I can get very easy out-of-box speech recognition. For styling I chose Joy UI, a variant of Material UI (I tend to prefer Material and it's kind because I am bad at styling and these frameworks make it easy).

Because there would be a UI component, I opted to develop my Rails APIs in GraphQL, as Apollo's JS client library is incredibly simple to use.

After deciding to add TTS, I had to make a few last-minute additions. ElevenLabs is my preferred TTS service, although it is rather expensive so I may explore other options (on a $22/month plan, I burned through more than half my quota in one stream - compared to $0.25 for OpenAI in the same stream). 

TTS processing is also more time-consuming, and I didn't want to risk a GQL request timing out in the middle, so I added background job processing via Sidekiq (using `foreman` to manage multiple Rails processes).

Making the TTS audio available in the UI was straightforward, I ended up using the built-in HTML5 `<audio>` wrapper which provides nice controls with little fuss. It's ugly and doesn't match the Material UI feel of the rest of the app, but it was good for a quick implementation. 

Getting the MP3 files to actually be available was very troublesome. I didn't want to bother with a ton of remote file hosting because I was only running this app locally. Browsers can access local system files (I setup the Sidekiq job to write to a local shared directory), so I assumed it would be easy to hook up, but React only has context for files included in `public`. I ended up symlinking the directory. [Heaven forgive me](https://xkcd.com/292/).

Notifying the front-end of finished job processing briefly sent me down the rabbit hole of GraphQL subscriptions - the React UI could subscribe to a particular query and update in realtime as the backend publishes updates. Unfortunately, the only subscription-management plumbing for Ruby GraphQL that is in the non-pro plan uses `ActionCable` which cannot handle publishing updates from background jobs (the two options that enable that use the DB and Cache, and are only available on Pro). I ended up going with the simple solution, setting `pollingInterval` in the client query to regularly refetch the message data.

## Usage Lessons

Barring a few small UI bugs, the maiden voyage of the app itself was fairly smooth with just a few complaints.

### Sidekiq Web UI

In the last-minute implementation rush I skipped adding Sidekiq UI. This made it more difficult to see/diagnose issues with the ElevenLabs integration. Remember kids, this is why proper log search systems and observability are important.

### Prompts

Not surprising to anyone with more ChatGPT experience than me, getting the right system message is tricky. It took several tries to get something with the tone that I wanted, but thankfully updating the system message is fairly easy via the UI; so I am glad I took time to build that out.

### Temperature

This was my biggest hope for the tool - introducing more variability - but it ended up breaking in ways I did not expect. Usually the output was not random enough, but sometimes it was **too** random:

![Screenshot of ChatGPT's high-temperature response with a nonsensical beginning, quickly descending into pseudocode gibberish"](../../../assets/images/chatgpt_high_temperature.png)

This massively confused me on stream until I figured out it was temperature-related. I ended up removing the variable temperature option from the frontend for now. I might add it back in the future with a more limited range.

## Future Enhancements

I'm very happy with how it turned out - for a tool intended to be used by a single streamer, it does the job well and required minimal live debugging. I do still have a few features I might want to add:

* Explore moving OpenAI calls to the background
  * I did not initially include Sidekiq, but now that it's there I should explore this a bit more
  * The biggest factor will be async error handling - with the OpenAI calls synchronous, it provides me with more realtime feedback about anything that might go wrong; if it was async, I would want more handling to surface those errors in an easy-to-debug way (although maybe adding the Sidekiq web UI would be enough?)
* Adding "summary messages"
  * Playing Hitman, we can complete most levels in under an hour, even with ChatGPT's antics; but for longer games (i.e. Baldur's Gate 3) I'm expecting multiple sessions that will outlast the maximum message context of 4,096 tokens
  * I'd like to build a summarization method - regularly send history to ChatGPT and ask it to summarize and rank importance (that way ChatGPT is in charge of deciding what it wants to remember), and including the most important summaries in the system message
  * This is something I began building but abandoned as a non-MVP feature
* Small UI tweaks
  * Clean up styling
  * Make the Message tab more messenger-like (visually)
  * Use a better-styled audio player for TTS
* Revisit temperature
  * Reduce the variable range - minimum of the default used by ChatGPT (`1`), maximum ???? (maybe around 1.5?)
  * Variable temp value is currently untracked - store in DB and show in UI for debugging
