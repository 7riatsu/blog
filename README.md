# Local development

```sh
hugo server -D
```

Access to http://localhost:1313/

# Add content

```sh
hugo new content content/post/{something}/index.md
```

After creating a new content, you need to edit file header

```markdown
---
title: "something"
date: 2024-12-27T20:13:06+09:00
author: "Atsuki Narita"
description: "write something"
tags: []
draft: false
share: true
---
```

# Release

```sh
# build
hugo

# commit and push
git push
```

# References
[HUGO Quick start](https://gohugo.io/getting-started/quick-start/)
