Tech Talks
---

This project uses [mkslides](https://github.com/MartenBE/mkslides) to render markdown files into [reveal.js](https://revealjs.com/) slideshows.

Requires `uv`.

To get started:

- Create a new markdown file in `src/`
- Or, create a folder in `src/` and add a Markdown file with the same name to it
- Build out slidedeck (see existing decks, use references below)
- Run `mkslides`:
```shell
uv run mkslides serve src/<optional folder path>
```
- Navigate to http://localhost:8000
- Use left/right (or `n`/`p`) to navigate slides. Use `s` to open the speaker view.

References:

- Markdown examples can be found here: https://github.com/HoGentTIN/hogent-markdown-slide
- Also reference reveal.js docs
