# Repository Guidelines

This repository contains Zenn content. Keep article edits focused, readable, and aligned with Zenn rendering constraints.

## Content Layout

- Articles live in `articles/`.
- Books live in `books/`.
- Images live in `images/`.
- Working notes and reusable writing guidance live in `notes/memo.md`.

For article-specific images, prefer a folder named after the article slug:

```text
images/<article-slug>/
```

Reference images from Markdown with absolute Zenn paths:

```markdown
![](/images/<article-slug>/<file>.png)
```

## Writing Style

- Start from the concrete problem, confusion, or field experience.
- Preserve the author's point of view and decision process.
- Avoid turning drafts into generic best-practice articles.
- Use natural Japanese. It is fine to keep phrases like `めんどくさい`, `つらい`, `ありがたい`, and `しっくりきている` when they match the article.
- Do not overstate certainty. Prefer grounded phrasing such as `自分の場合は`, `今は`, `良さそう`, or `という感覚です` when appropriate.

## Diagram Guidance

- Use diagrams to help readers grasp structure, not to replace the article text.
- Keep the diagram title short.
- Avoid long explanatory subtitles directly under the title.
- Keep useful labels inside boxes and around arrows so the diagram can still be read on its own.
- Match diagram labels to the article's actual wording.
- Do not include work that did not actually happen.
- Check that text does not overflow boxes after rendering.

Prefer PNG or JPEG references in Markdown. SVG files may remain as editable source files, but article Markdown should usually point to generated PNG/JPEG files for compatibility.

When converting SVG diagrams, verify the generated raster image visually. Some converters create square thumbnails or change the apparent spacing; regenerate with a tool that preserves the SVG aspect ratio when needed.

For more concrete accumulated notes, check `notes/memo.md`. Add future learnings there under a dated heading instead of creating topic-specific note folders.

## Editing Rules

- Keep changes scoped to the requested article or note.
- Do not rewrite unrelated sections just to polish them.
- When adding images, keep source assets and rendered assets together.
- After changing image references, verify that referenced files exist.
- Before finishing, check for obvious Markdown issues, stale references, and terminology drift between text and diagrams.
