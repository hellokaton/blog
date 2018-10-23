---
layout: page
title: 友链
permalink: links.html
image: /public/images/links.jpeg
order: 4
---

> God made relatives. Thank God we can choose our friends.

<ul class="links">
{% for item in site.links %}
<li>
    <p>
        <a href="{{ item.url }}" target="_blank" title="{{ site.title }}">
        {{ item.title }}
        </a>
    </p>
</li>
{% endfor %}
</ul>

经常更新博文可交换友链接，不定期对无法访问的网址进行清理，请保证自己的链接长期有效。