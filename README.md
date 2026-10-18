# mohamedmoatazsec.github.io

Source for [mohamedmoatazsec.github.io](https://mohamedmoatazsec.github.io): web & API
penetration testing write-ups, methodology notes and tooling posts.

Built with [Jekyll](https://jekyllrb.com) on GitHub Pages using a custom theme (no
remote theme, no build step beyond what Pages does itself). Dark by default, with a
light theme that follows the OS setting and a toggle that remembers the visitor's choice.

## Layout

```
_config.yml          site settings, permalinks, plugins, front-matter defaults
_layouts/            default (shell) · page · post
_includes/           head · header · footer · post-card
_posts/              published posts, YYYY-MM-DD-slug.md → /writeups/YYYY/slug/
_drafts/             not built on Pages; writeup-template.md is the starting point
assets/css/main.css  the whole theme, plain CSS, design tokens at the top
assets/js/theme.js   theme toggle
assets/img/          favicons and the social-sharing card
index.html           home · writeups.html → /writeups/ · about.md → /about/ · 404.html
```

## Writing a post

1. Copy `_drafts/writeup-template.md` to `_posts/YYYY-MM-DD-slug.md`.
2. Fill in the front matter. For a real finding, keep the `report:` block: it renders the
   summary panel on the post and the severity badge on the cards. Methodology or tooling
   posts can drop it and set `category:` instead.
3. Write the lede, then put `<!--more-->` after the first paragraph; everything above it is
   the excerpt.
4. Push to `main`. Pages rebuilds in about a minute.

Posts dated in the future are not built (`future: false`), so a post dated later today
will appear only after that time has passed in `Africa/Cairo`.

## Local preview

```sh
bundle install
bundle exec jekyll serve --livereload --drafts
```

Then open <http://127.0.0.1:4000>. The `github-pages` gem pins the same Jekyll and plugin
versions Pages uses, so what builds locally builds there.

## Licence

Content (posts, images) © Mohamed Moataz. Theme code (layouts, includes, CSS, JS) is
MIT-licensed; reuse it, just do not reuse the posts.
