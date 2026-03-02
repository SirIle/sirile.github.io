---
title: "{{ replaceRE `^\d{4}-\d{2}-\d{2}-` `` .File.ContentBaseName | replaceRE `-` ` ` | humanize }}"
date: {{ .Date }}
draft: true
tags: []
ShowToc: true
TocOpen: false
---

