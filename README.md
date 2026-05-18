# Kamil Wojtczyk

Personal blog for [kamilwojtczyk.com](https://kamilwojtczyk.com), built with [Astro](https://astro.build) and Markdown content files.

The blog is for notes about code, coffee, developer platforms, infrastructure, and personal projects.

## Writing

Posts live in `posts/` as Markdown files. Each post needs frontmatter with `title`, `slug`, `description`, `tags`, and `added`.

## Development

All commands are run from the root of the project:

| Command                 | Action                                    |
| :---------------------- | :---------------------------------------- |
| `corepack pnpm i`       | Installs dependencies                     |
| `corepack pnpm dev`     | Starts the dev server at `localhost:4321` |
| `corepack pnpm build`   | Builds the production site to `./dist/`   |
| `corepack pnpm preview` | Previews the production build locally     |

## Credits

Based on [cassidoo/blahg](https://github.com/cassidoo/blahg).
