# hedgedb.github.io

Source for the HedgeDB landing page, built with [Sphinx](https://www.sphinx-doc.org/)
and the [Furo](https://pradyunsg.me/furo/) theme.

## Build locally

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

make html         # output in _build/html
make serve        # builds + serves on http://localhost:8000
```

## Deploy

Pushing to `main` triggers `.github/workflows/pages.yml`, which builds the
site and publishes it via GitHub Pages.

To enable it on a fresh repo, in **Settings → Pages**, set the source to
**GitHub Actions**.

## Layout

```
.
├── conf.py                 Sphinx config
├── index.md                Landing page
├── architecture.md         Design deep-dive
├── getting-started.md      Build & API examples
├── _static/                Logo, diagrams, custom.css
├── requirements.txt        Sphinx + Furo + myst-parser
├── Makefile                Standard Sphinx make targets
└── .github/workflows/      Pages deploy
```

Content is authored in Markdown via `myst-parser`. To edit, just change the
`.md` files and run `make html`.
