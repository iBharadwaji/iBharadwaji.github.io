---
layout: page
title: "Interview Prep"
permalink: /interview-prep/
---

{% assign all = site.pages | where_exp: "p", "p.qsection" %}{% assign idx = site.pages | where_exp: "p", "p.qindex" | sort: "qname" %}Interview questions worked through and recorded for revision. {{ all | size }} so far.

## Sections

{% for s in idx %}{% assign n = all | where: "qsection", s.qindex | size %}- [{{ s.qname }}]({{ s.permalink }}) — {{ n }}
{% endfor %}

## Weak spots

Anything not yet solid. Start here before an interview.

{% assign weak = all | where_exp: "p", "p.confidence != 'solid'" | sort: "confidence" %}{% if weak.size == 0 %}Nothing outstanding.{% else %}{% for q in weak %}- [{{ q.title }}]({{ q.permalink }}) — `{{ q.confidence }}` — {{ q.qsection }}
{% endfor %}{% endif %}
