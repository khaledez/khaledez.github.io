# khaledez.net

A Hugo site with no theme: the templates are in `layouts/` and the one stylesheet is `assets/css/main.css`. Pushing to `main` deploys it through `.github/workflows/hugo.yml`.

- Preview: `hugo server`, then open http://localhost:1313
- New post: `hugo new content blog/<slug>.md`, then write Markdown. A post appears once its `date` has passed and it has no `draft: true`.
- Front matter a post can use: `title`, `date`, `description` (shown under the title and in lists), `kicker` (the small line above the title), and `cast` (a list of `name`/`role`).
- Shortcodes for long posts: `{{< part id="" kicker="" title="" >}}…{{< /part >}}`, `{{< stamp >}}…{{< /stamp >}}` (a chapter's date and place) and `{{< lesson >}}…{{< /lesson >}}`. Each `###` heading in a post is a numbered chapter.
- Pages that predate Hugo (`hijridate.html`, `islamic-civ-research.html`) are served from `static/` as they were.
