# InfraAI Research Blog

Source for the research blog of the [InfraAI Center](https://infraai.center) at The University of Texas at Austin.

The blog features work by InfraAI students and researchers, including:

- Ideas, experimental lessons, and interesting tangents that do not fit a standard research paper.
- Accessible explanations of research papers, with additional context and technical detail.

The site uses Jekyll with custom layouts and CSS. GitHub Pages builds and deploys
the site when changes are pushed to `main`.

## Local preview

With Ruby and Bundler installed:

```sh
bundle install
bundle exec jekyll serve --host 127.0.0.1 --port 4000
```

Open <http://localhost:4000>. Jekyll rebuilds the site when its source files change.

## Publishing a post

Copy `templates/post.md` to `_posts/YYYY-MM-DD-short-title.md`, fill in its
metadata, and write the post in Markdown. Posts appear on the landing page
automatically, newest first; no manual homepage tile is needed.
For work in progress, use `_drafts/short-title.md`
and preview with `bundle exec jekyll serve --drafts`.

The title, author, publication date, description, and tags are shown by the post
layout. Cover images and an original-source attribution are optional. Code
blocks, tables, figures, and footnotes use the shared article styles.

`_data/external_posts.yml` lists articles hosted elsewhere. Remove an external
entry when its edited version is published here to avoid a duplicate.

## Editing the design

- `index.html`: landing page.
- `_layouts/`: shared page and article layouts.
- `_includes/post-card.html`: an article in the landing-page list.
- `assets/css/site.css`: responsive styles and article typography.
- `assets/images/`: covers and the text-based favicon. Do not add images that
  contain the UTexas logo.

Review changes on localhost before pushing. Push only after Rohit's explicit OK.
