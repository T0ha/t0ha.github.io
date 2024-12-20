---
layout: page
title: Contacts 
published: true
permalink: /contacts/
contacts:
  email: t0hashvein@gmail.com
  social:
    - platform: twitter
      title: "@t0ha666"
      user_url: https://twitter.com/T0ha666
    - platform: bluesky
      title: "@t0ha.ru"
      user_url: https://bsky.app/profile/t0ha.ru
    - platform: linkedin
      title: Anton Shvein
      user_url: https://www.linkedin.com/in/anton-shvein/ 
    - platform: mastodon
      title: "@t0ha@mastodon.social"
      user_url: https://mastodon.social/@t0ha
  other:
    - platform: github
      title: T0ha
      user_url: https://github.com/T0ha
    - platform: stackoverflow
      title: T0ha
      user_url: https://stackoverflow.com/users/3489550/t0ha
    - platform: youtube
      title: Anton Shvein
      user_url: http://www.youtube.com/@AntonShvein
    - platform: twitch
      title: war1and
      user_url: https://www.twitch.tv/war1and
  messanger:
    - platform: telegram
      title: "@war1and"
      user_url: https://t.me/war1and
  certs:
    - platform: ssh
      title: Publick Key
      user_url: "/assets/ssh.pub"
    - platform: GPG
      title: Publick Key
      user_url: "/assets/gpg.pub"
---

## Geo
- Location: Perm, Russia
- Time zone: UTC+5

## Email
<t0hashvein@gmail.com>

## Message Me
{% for entry in page.contacts.messanger %}
- {{entry.platform | capitalize }}: [{{ entry.title | default: entry.user_url }}]({{entry.user_url}})
{%- endfor %}

## Social media

{% for entry in page.contacts.social %}
- {{entry.platform | capitalize }}: [{{ entry.title | default: entry.user_url }}]({{entry.user_url}})
{%- endfor %}

## Other

{% for entry in page.contacts.other %}
- {{entry.platform | capitalize }}: [{{ entry.title | default: entry.user_url }}]({{entry.user_url}})
{%- endfor %}

## Certificates

{% for entry in page.contacts.certs %}
- {{entry.platform | capitalize }}: [{{ entry.title | default: entry.user_url }}]({{entry.user_url}})
{%- endfor %}
