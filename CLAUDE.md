# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**traliant-claude** — a Python project in early initialization. No source code or build infrastructure exists yet beyond a `.gitignore`.

## Expected Toolchain

Based on the `.gitignore`, the project is expected to use:

- **Testing:** `pytest`
- **Linting/Formatting:** `ruff`
- **Type checking:** `mypy`
- **Package management:** likely `uv`, `poetry`, `pipenv`, or `pdm` (all included in `.gitignore`)

Update this file once the toolchain is confirmed and add the actual commands (e.g., `uv run pytest`, `ruff check .`, `mypy .`).
