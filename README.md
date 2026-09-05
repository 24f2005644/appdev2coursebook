# MAD2 Course Book

Course reference site for Modern Application Development 2, built with [MkDocs](https://www.mkdocs.org/) and [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

## Project structure

```text
docs/
├── index.md
├── week1/
├── week2/
├── ...
└── week12/
mkdocs.yml
requirements.txt
```

Each week should contain an `index.md` overview and its topic Markdown files. Store images and other media in an `assets/` folder inside the relevant week.

## Run locally

Prerequisites: Python 3.9 or newer.

```bash
python -m venv .venv
```

Activate the virtual environment:

- Windows PowerShell: `.venv\Scripts\Activate.ps1`
- macOS/Linux: `source .venv/bin/activate`

Install dependencies and start the preview server:

```bash
python -m pip install -r requirements.txt
mkdocs serve
```

Open <http://127.0.0.1:8000> in your browser. The server reloads when Markdown or configuration files change.

## Build the site

```bash
mkdocs build --strict
```

The generated site is written to `site/` and is not committed.

## Adding course content

1. Add or edit Markdown files under the appropriate `docs/weekN/` folder.
2. Add new pages to the corresponding week in `mkdocs.yml`.
3. Run `mkdocs build --strict` locally.
4. Open a pull request with the content change.
