---
layout: home
---

# SnapyHook Research Notes

Systems, Reverse Engineering, and Runtime Analysis.

## 📄 Research Posts

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%b %d, %Y" }}
{% endfor %}
