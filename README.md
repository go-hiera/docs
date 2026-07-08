<p align="center"><img src="https://raw.githubusercontent.com/go-hiera/brand/main/social/go-hiera.png" alt="go-hiera/docs" width="720"></p>

<!-- The org brand assets live in github.com/go-hiera/brand; the banner above
     renders once that repo is published. -->

# go-hiera/docs

Versioned documentation for [go-hiera](https://github.com/go-hiera), built with
[MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and versioned
with [mike](https://github.com/jimporter/mike). Published to the `gh-pages`
branch and served at <https://go-hiera.github.io/docs/>.

The organization landing page ([go-hiera.github.io](https://go-hiera.github.io))
links here.

## Local preview

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
mkdocs serve                       # http://localhost:8000 (current sources)
mike serve                         # preview the versioned site
```

## Releasing a new docs version

```bash
mike deploy --push --update-aliases <version> latest
mike set-default --push latest
```
