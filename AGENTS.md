# AGENTS.md

PDF Translator for Human: a Streamlit web app (`app.py`) and CLI (`translator_cli.py`) that
translate PDFs per-page, side by side with the original. It embeds a **vendored copy** of
`deep-translator` in `deep_translator/` with project-specific modifications.

## Architecture

- `app.py` — Streamlit entrypoint (`streamlit run app.py`). Two-layer caching: page-level PDF
  cache in `.cached/<sha256>.pdf` (key built in `get_cache_key`) plus block-level SQLite cache.
- `translator_cli.py` — offline CLI; writes translation as an optional-content (OCG) layer so the
  original stays toggleable. `--no-original` hides it. Uses `ChatGptTranslator` / `GoogleTranslator`.
- `pdf_translator/core.py` — shared block-rewriting primitives (`get_blocks`, `write_translated_page`)
  used by both app and CLI. Keep all rendering logic here, not in the entrypoints.
- `translation_cache.py` — block-level SQLite cache (`.cached/translations.sqlite3`). Keyed by
  translator/source/target/model/text-hash.
- `deep_translator/openai_compatible.py` — `OpenAICompatibleTranslator` (used for local LLMs, ChatGPT,
  DeepSeek, etc.). Adds `translate_batch`, 3-retry with exponential backoff, and raises
  `TranslationFailed`; `app.py` falls back per-block on `TranslationFailed`.

## Dependencies and install gotchas

- `pyproject.toml`/`poetry.lock` are **stale upstream deep-translator metadata** (name
  `deep-translator`, version 1.11.4). They do NOT declare the app's runtime deps.
- The app needs `streamlit`, `pymupdf` (imported as `pymupdf`, not `fitz`), and `openai`.
  Install them with `pip install -r requirements.txt` or `pip install streamlit pymupdf openai`;
  `pip install -e .` / `poetry install` will not provide them.
- `deep_translator/` diverges from upstream (retry/batch in `chatgpt.py`, `openai_compatible.py`).
  Never blindly sync with the `upstream` remote (nidhaloff/deep-translator) — it will clobber these changes.
- Import-only works from the repo root: `app.py` does `import translation_cache` and
  `from pdf_translator import ...` (repo-root modules, not installed packages).

## Commands

- Run web app: `streamlit run app.py` (defaults to Google translator, no key; OpenAI-compatible
  defaults to `http://localhost:8080/v1` for local LLMs).
- Run CLI: `python translator_cli.py --source en --target zh-CN input.pdf`.
- Lint/format: `pre-commit run --all-files` (black 79, isort `--profile black`, flake8, pycln).
  flake8 config is in `.flake8` (max-line-length 109, ignores E203/W503/E402).
- Tests: `pytest tests/` — these are upstream deep-translator tests and mostly hit **live network
  translation services**; unreliable offline and unrelated to the PDF/web features. There is no test
  suite for `app.py`, `pdf_translator`, or the caches.

## Environment notes

- `.cached/` and `*.pdf` are gitignored — generated translations, the SQLite block cache, and page
  caches all live there and are disposable.
- `run_translator_web.sh` is a user bootstrap script that `git clone`s the public repo — do not run it
  inside this checkout.
