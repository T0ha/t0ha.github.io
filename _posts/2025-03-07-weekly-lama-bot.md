---
layout: post
locale: en_US
title: "Weekly Lama Bot: Working on Analytics"
image: /assets/images/weekly/20250307_lama_bot_user_engagement.png
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
{% picture "{{page.image}}" --alt Lama Bot Stats %}

I haven't done live streams and haven't written weekly updates for a couple of weeks, since I was travelling and the it was my birthday and so on.

But I was not doing nothing these weeks. 

First of all, on Feb, 11th 19 new users started to use [Lama Bot](https://t.lamabot.io) . it's a pity but I can't find a way to get refer or utm-tags from t.me link, so I can't really understand the source of new users. I was thinking they could come from [this post on X](https://x.com/T0ha666/status/1889202556309835957) but looks like it's not. 

These users generated a bunch of usage scenarios so I was busy with analysing behaviours and logs and got a couple of assumptions about how to make the bot better:
- Followup messages sending time should be adjusted after a couple of days according to users' "most active time". For most of users it could be easily determined statistically.
- Some users bans the bot due to annoying followups but it seems they just not its target audience.
- There are a couple of users (excluding me) who use Lama Bot for most of its "life". That's great and promising. It seems it (or he) has its market fit.
- Most of Gemini response, except message text, is useless.

Based on these and some other observations I decided to add the following tasks
- Add more complex scheduling algo for followups;
- Investigate followup response rates and adjust accordingly.
- Add a bunch of new followups with practices.
- Start investigation of native language processing to make practice sugestion algo a bit more smart.
- Dig myself deeper in psychological safety and ethics to make the bot more comfortable for users.


Next Tuesday (2025-03-11) at 10:00 UTC we are going to continue working on landing page. 
See you on [Twitch](https://www.twitch.tv/war1and) and [YouTube](https://youtube.com/live/ajYG5RLPb-Q?feature=share).