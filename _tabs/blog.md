---
icon: fas fa-pen-fancy
order: 4
title: Blog
hide: false
---

## Blog & Research Articles

I write about molecular dynamics, chemical engineering, scientific computing, and research methodology. 

**Focus Areas**:
- Molecular dynamics simulation techniques and best practices
- Force field development and validation
- Data analysis and scientific computing with Python
- HPC and performance optimization
- Research tutorials and technical deep-dives
- Scientific programming and open-source development

---

## Recent Articles

<div class="article-list">
{% for post in site.posts %}

### [{{ post.title }}]({{ post.url }})

**Date**: {{ post.date | date: "%B %d, %Y" }}

**Categories**: {% for category in post.categories %}<span class="category">{{ category }}</span> {% endfor %}

**Tags**: {% for tag in post.tags %}<span class="tag">#{{ tag }}</span> {% endfor %}

{{ post.description | default: post.content | strip_html | truncatewords: 50 }}

[Read More →]({{ post.url }})

---

{% endfor %}
</div>

---

## Articles by Category

{% for category in site.categories %}
### {{ category[0] }}

{% for post in category[1] %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

{% endfor %}

---

## Articles by Tag

{% for tag in site.tags %}
- **{{ tag[0] }}** ({{ tag[1].size }} articles)
{% endfor %}

---

## Subscribe

To stay updated with new articles, you can:
- Follow my [Twitter](https://twitter.com/alokranjancheme)
- Follow my [GitHub](https://github.com/alokranjancheme)
- Check back here regularly

---

## Comments & Discussion

Have thoughts on an article? [Contact me](/contact/) to discuss or collaborate.

