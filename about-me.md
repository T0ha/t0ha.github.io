---
layout: page
title: Who am I?
published: true
permalink: /
---
![Anton Shvein](/assets/images/t0ha.jpeg){: style="display: block; width: 25%; float: left; margin-right: 20px; margin-bottom: 20px;" }

👤 I'm Anton Shvein. I'm 38. My other nicknames are T0ha or war1and.

🌐 I live in Perm but travel a lot. 

🧑‍💻 I'm a software developer for more than a dozen years. Currently my main focus is full-stack web and chat-bot development in [Erlang](http://erlang.org) and [Elixir](https://elixir-lang.org). I use [Phoenix web framework](https://phoenixframework.org) a lot in my daily job.

🧑‍🧑‍🧒‍🧒 I have a wife and 2 sons.

📽️ I livestream my work on [Twitch](https://www.twitch.tv/war1and) and run a  [YouTube channel](https://www.youtube.com/c/AntonShvein). 

✍️ I write to this [blog]({% link index.md %}).

📿 I'm a Mahayana Buddhist.

🗣️ My mother tongue is Russian. I communicate in English as well. Also I studied French and Latin but so so. Learning Tibetan (བོད་སྐད་)

⚗️ I'm Master of Biology.

### Best Articles

{% for post in site.posts %}
{% unless post.categories contains "weekly" %}
- [{{ post.title }}]({{ post.url }})
{% endunless %}
{% endfor %}
