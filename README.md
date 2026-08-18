# GitHub Pages files

This folder is ready to use as a simple GitHub Pages/Jekyll site.

Files:

- `index.md` — the editable report source.
- `_layouts/default.html` — the HTML shell Jekyll wraps around the Markdown.
- `assets/css/styles.css` — all page styling.
- `_config.yml` — minimal Jekyll configuration.

The stylesheet is loaded by this line in `_layouts/default.html`:

```html
<link rel="stylesheet" href="{{ '/assets/css/styles.css' | relative_url }}">
```

That `relative_url` filter is important for project Pages sites because the site may live under
`https://USERNAME.github.io/REPOSITORY/` rather than at the domain root.
