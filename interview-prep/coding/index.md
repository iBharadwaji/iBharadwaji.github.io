---
layout: page
title: "Interview Prep — Coding"
permalink: /interview-prep/coding/
qindex: coding
qname: "Coding"
---

Algorithms and data structures, in any language. {% assign qs = site.pages | where: "qsection", "coding" | sort: "title" %}{{ qs | size }} questions. [Cram view](/interview-prep/coding/cram/)

| Question | Confidence | Difficulty | Tags |
|---|---|---|---|
{% for q in qs %}| [{{ q.title }}]({{ q.permalink }}) | {{ q.confidence }} | {{ q.difficulty }} | {{ q.tags | join: ", " }} |
{% endfor %}

[All sections](/interview-prep/)
