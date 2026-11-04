# Diagrams

The architecture docs use hand-laid-out SVGs instead of Mermaid, so each diagram fits a docs column without wasted space

- Every SVG embeds [diagram-style.css](diagram-style.css) verbatim in one `<style>` element. It is based on the litellm-docs theme (Geist and Geist Mono, the accent `#0117be`, its greys) and switches to the dark palette with `prefers-color-scheme`
- Use only its classes, with no inline styles or hard-coded colors. `box core` is the Rust router core, `box host` the host and Python side, `box store` caches and Redis, `box warn` errors, `box ok` results, `box decision` a question
- Keep the `viewBox` at most 880 wide and as short as the content allows, start with the `canvas` rect, and give the root `role="img"` and a `<title>`
- Edit a diagram by changing its SVG directly. Diagrams for a page live in an `img/` folder next to it
