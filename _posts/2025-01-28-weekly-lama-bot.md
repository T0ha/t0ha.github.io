---
layout: post
locale: en_US
title: "Weekly Lama Bot: Making Followup Messages Less Annoying"
youtubeurl: https://youtu.be/Iur9XAltfHA
tags:
  - buildinpublic
  - chatbot
  - ai
  - llm
  - telegram
  - youtube
  - video
  - livestream
  - lamabot
  - elixir
  - phoenix
  - phoenix-framework
  - nvim
status: published
categories:
  - en
  - buildinpublic
  - lamabot
published: true
---
{% youtube page.youtubeurl %}


Thank you, dear friends and followers, for an amazing livestream this Tuesday. It was my first really LIVE stream with engagement in the chat and feedback from you. I'm really excited! 😊

We started with an attempt to integrate ObanWeb into the project. It was recently opensourced. But we failed  due to outdated dependencies. So we gave it up for a while until deps update.

Then we fixed logic of followup messages to send a message only if user haven't communicated with the bot for a day as well as skip followup if previous one was not responded.

Since I've never seen anyone watching the stream on YouTube I decided to give it up. For now I'm streaming only on Twitch.

So, next Tuesday (2025-02-04) at 10:00 UTC we are going to work on code quality, including credo, dialyzer and tests. This will prepare the project for Phoenix update with less issues.

See you on [Twitch](https://www.twitch.tv/war1and).