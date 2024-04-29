---
title: Packaging Errbot Plugins with Poetry
subtitle: Using Poetry to package an Errbot backend
date: 2024-04-28T22:23:30-05:00
slug: 2d16883
draft: false
author: Andrew
description: 
keywords:
comments: false
weight: 0
tags:
  - errbot
  - chatbot
  - poetry
categories:
  - open-source
  - python
summary:
hideMeta: false
hideSummary: false
showtoc: false
tocopen: false
ShowPostNavLinks: false
ShowBreadCrumbs: false
ShowWordCount: false
hideFooter: false

---

Errbot's documentation has [information on packaging](https://errbot.readthedocs.io/en/latest/user_guide/plugin_development/basics.html#packaging) 
for plugins and backends using setup.py.  I'm a fan of using Poetry for my python projects, so I wanted to figure out how to make that work for my Errbot plugins and backends.

<!--more-->

It ended up being more simple than I expected. Poetry has a feature to [declare plugins](https://python-poetry.org/docs/plugins/), but the docs are for
creating a plugin for poetry itself. There is a [github issue](https://github.com/python-poetry/poetry/issues/658) about clarifying in the docs, but you 
can use this config for making anything that uses setup.py entry_points for plugins! For errbot, this looks something like this:

```code
[tool.poetry.plugins."errbot.backend_plugins"]
{module} = "{python_module}:{backend class}"
```

So, for my err-aprs-backend, it ends up with:

```code
[tool.poetry.plugins."errbot.backend_plugins"]
aprs = "aprs_backend:APRSBackend"
```

Which matches up with my aprs.plug

```code
[Core]
Name = APRS
Module = aprs

[Documentation]
Description = Backend for APRS
```

You will also need to add your plug file to your include so it is packaged. I did this with:

```code
include = ["*.plug"]
```

The full [pyproject.toml](https://github.com/andrewthetechie/err-aprs-backend/blob/main/pyproject.toml) for err-aprs-backend can be found on [Github](https://github.com/andrewthetechie/err-aprs-backend).