---
layout: default
---

# SnapyHook Research Notes

**Systems · Reverse Engineering · Runtime Analysis**

This site documents my independent research into how large native applications behave internally —  
with a focus on **binary analysis, virtual machines, scripting runtimes, and execution pipelines**.

The write-ups here are derived from hands-on analysis using static disassembly, runtime instrumentation,
and controlled experimentation.

---

## 📄 Research Posts

{% for post in site.posts %}
- **[{{ post.title }}]({{ post.url }})**  
  <span style="color: #9f9f9f;">{{ post.date | date: "%B %d, %Y" }}</span>
{% endfor %}

---

## 🔗 Links

- GitHub: https://github.com/SnapyHook  
- LinkedIn: https://www.linkedin.com/in/prashar-aryan/
