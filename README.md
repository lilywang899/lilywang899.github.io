# Personal site + blog (GitHub Pages)

A minimal Jekyll site. GitHub Pages builds it automatically, so you do not need Ruby installed.

## Publish (one time)

1. Edit `_config.yml`: set `title`, `author`, `email`, and replace `USERNAME` (two places) with your GitHub username.
2. On GitHub, create a **new public repository** named exactly `USERNAME.github.io` (your username, lowercase). Do not add a README.
3. In this folder:
   ```
   git add .
   git commit -m "Initial site"
   git remote add origin https://github.com/USERNAME/USERNAME.github.io.git
   git push -u origin main
   ```
4. On GitHub: repo **Settings > Pages**, make sure Source is "Deploy from a branch", branch `main`, folder `/ (root)`.
5. Wait a minute, then visit `https://USERNAME.github.io`.

## Write a post

Create `_posts/YYYY-MM-DD-your-title.md`:

```markdown
---
title: "Your title"
date: 2026-11-06 10:00:00 -0800
tags: [example]
---

Intro paragraph (shows as the excerpt).

<!--more-->

Rest of the post.
```

Commit and push. The site rebuilds in about a minute.

## Pages

- `index.md` - home page (shows the 5 most recent posts)
- `blog/index.html` - all posts grouped by year
- `about.md` - about page
- `_layouts/` - HTML templates, `assets/css/style.css` - styles

## Local preview (optional)

Requires Ruby. Then:

```
bundle install
bundle exec jekyll serve
```

and open http://localhost:4000.
