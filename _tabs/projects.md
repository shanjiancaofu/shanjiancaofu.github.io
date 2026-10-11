---
layout: page
title: 项目
icon: fas fa-code-branch
order: 3
---

# 项目

这里收录正在学习、维护和持续整理的项目。项目页保留项目方向、技术栈和入口，具体实现细节会继续沉淀到博客文章中。

{% for project in site.data.projects %}
<article class="project-page-card">
  <div class="project-page-card__meta">{{ project.type }} · {{ project.status }}</div>
  <h2>{{ project.name }}</h2>
  <p>{{ project.description }}</p>
  <div class="project-page-card__stack">
    {% for item in project.stack %}<span>{{ item }}</span>{% endfor %}
  </div>
  <a href="{{ project.url }}" target="_blank" rel="noopener noreferrer">查看 GitHub 仓库 <i class="fas fa-arrow-up-right-from-square"></i></a>
</article>
{% endfor %}
