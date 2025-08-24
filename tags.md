---
layout: page
titles:
  # @start locale config
  en      : &EN       Tags
  en-GB   : *EN
  en-US   : *EN
  en-CA   : *EN
  en-AU   : *EN
  zh-Hans : &ZH_HANS  标签
  zh      : *ZH_HANS
  zh-CN   : *ZH_HANS
  zh-SG   : *ZH_HANS
  zh-Hant : &ZH_HANT  標籤
  zh-TW   : *ZH_HANT
  zh-HK   : *ZH_HANT
  ko      : &KO       태그
  ko-KR   : *KO
  fr      : &FR       Étiquettes
  fr-BE   : *FR
  fr-CA   : *FR
  fr-CH   : *FR
  fr-FR   : *FR
  fr-LU   : *FR
  # @end locale config
key: page-tags
---

## Tags

{% for tag in site.tags %}
  {% capture tag_name %}{{ tag | first }}{% endcapture %}
  <h3 id="{{ tag_name | slugify }}">{{ tag_name }}</h3>
  {% for post in tag.last %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
{% endfor %}
