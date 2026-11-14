# Article rules

These rules apply when creating or editing articles in `content/posts/`.

## Images

- All images in an article must be JPEG files with the `.jpg` extension. Do not use SVG, PNG, WebP, or GIF.
- If an image is produced in another format (for example an SVG diagram), convert it to `.jpg` before adding it to the article.
- Store all of an article's images in an `img` folder in the article root: `content/posts/<slug>/img/`. Do not put images next to `index.md` or anywhere else.
- Reference them with a relative path including the folder, e.g. `![Alt text](img/image-01.jpg)`.
- Naming: `image-NN.jpg` for illustrations, `diagram-<topic>.jpg` for diagrams.
- Every image needs descriptive alt text.

## Tables

- Do not use Markdown or HTML tables in articles.
- When content would naturally be a table (comparisons, before/after numbers, option matrices), present it as an image instead: create a diagram, convert it to `.jpg`, and add it following the image rules above (e.g. `diagram-<topic>.jpg`).
- The alt text of such an image must carry the key data from the table, so the information stays accessible.
