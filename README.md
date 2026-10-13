# weronikagutfeter.github.io

Personal site built with [Jekyll](https://jekyllrb.com/) and the [Jasper](https://github.com/jekyllt/jasper) theme (Ghost Casper port).

## Local development

With Docker:

```bash
docker run --rm -v "$PWD:/srv/jekyll" -w /srv/jekyll -p 4000:4000 jekyll/jekyll:3.8 \
  sh -c "bundle install && bundle exec jekyll serve --host 0.0.0.0"
```

Or with a local Ruby toolchain:

```bash
bundle install
bundle exec jekyll serve
```

Then open http://localhost:4000/

## Deployment

Pushing to `master` runs GitHub Actions, which builds the site and publishes it to the `gh-pages` branch.

In the repo **Settings → Pages**, set the source to the **gh-pages** branch (root).
