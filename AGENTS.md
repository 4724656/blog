## Development

When starting the dev server, use background mode:

```
astro dev --background
```

Manage the background server with `astro dev stop`, `astro dev status`, and `astro dev logs`.

## Documentation

Full documentation: https://docs.astro.build

Consult these guides before working on related tasks:

- [Adding pages, dynamic routes, or middleware](https://docs.astro.build/en/guides/routing/)
- [Working with Astro components](https://docs.astro.build/en/basics/astro-components/)
- [Using React, Vue, Svelte, or other framework components](https://docs.astro.build/en/guides/framework-components/)
- [Adding or managing content](https://docs.astro.build/en/guides/content-collections/)
- [Adding styles or using Tailwind](https://docs.astro.build/en/guides/styling/)
- [Supporting multiple languages](https://docs.astro.build/en/guides/internationalization/)

## Publishing Content / Writing Posts (for OpenClaw & AI Agents)

When writing or generating blog posts for this site, strictly follow these rules:

1. **Target Directory**: All posts MUST be created in `content/posts/`.
2. **File Naming**:
   - Use English kebab-case, short pinyin, or date-prefixed alphanumeric slugs (e.g. `2026-08-25-my-post.md`, `docker-guide.md`, `thoughts-01.md`).
   - NEVER use Chinese characters or spaces in filenames to avoid messy URL percent-encoding.
3. **Required Frontmatter Format**:
   ```yaml
   ---
   title: "文章的中文完整标题"
   description: "文章导语/摘要（50-100字），用于列表展示和文章头部引用"
   publishDate: "YYYY-MM-DD"
   tags: ["tag1", "tag2"]
   ---
   ```
4. **Typography & Formatting**:
   - Use `##` for section headers (h2).
   - Use `>` for key quotes, highlights, or summaries.
   - Use `**bold**` for strong emphasis.
   - Keep Chinese typesetting clean with natural paragraph spacing.
5. **No Layout Alterations**: Do NOT modify any components, layouts, or CSS in `src/`. Only create/edit `.md` files in `content/posts/`.
