---
layout: page
titles:
  # @start locale config
  en      : &EN       Archive
  en-GB   : *EN
  en-US   : *EN
  en-CA   : *EN
  en-AU   : *EN
  zh-Hans : &ZH_HANS  归档
  zh      : *ZH_HANS
  zh-CN   : *ZH_HANS
  zh-SG   : *ZH_HANS
  zh-Hant : &ZH_HANT  歸檔
  zh-TW   : *ZH_HANT
  zh-HK   : *ZH_HANT
  ko      : &KO       아카이브
  ko-KR   : *KO
  fr      : &FR       Archives
  fr-BE   : *FR
  fr-CA   : *FR
  fr-CH   : *FR
  fr-FR   : *FR
  fr-LU   : *FR
  # @end locale config
key: page-archive
---

## Posts Archive

{% for post in site.posts %}
  {% assign currentdate = post.date | date: "%Y" %}
  {% if currentdate != date %}
    {% unless forloop.first %}</ul>{% endunless %}
    <h3 id="y{{post.date | date: "%Y"}}">{{currentdate}}</h3>
    <ul>
    {% assign date = currentdate %}
  {% endif %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> - {{ post.date | date: "%B %d" }}</li>
  {% if forloop.last %}</ul>{% endif %}
{% endfor %}