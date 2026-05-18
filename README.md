# Cassidy's blog template

[![Netlify Status](https://api.netlify.com/api/v1/badges/eab04209-5f7f-41ed-a8dd-c45a9ebb1834/deploy-status)](https://app.netlify.com/sites/blahg/deploys)

Hello, welcome. This is a blog ("blahg" is the proper spelling for Chicagoans) template. It's built with [Astro](https://astro.build), and content is edited directly as Markdown.

![cover](https://github.com/cassidoo/blahg/assets/1454517/b56ff04f-9499-48e7-be62-d9b422c4287d)

## See the blahg

[blahg.netlify.app](https://blahg.netlify.app/)

## To use the template

- Connect to your chosen hosting provider (see Deploy to Netlify button below if you want to go that route, otherwise use the GitHub template button above and pick a different one)
- Update `astro.config.mjs` with your domain
- Edit `src/settings/settings.json`
- Set your Twitter creator handle in `src/components/BaseHead.astro` (`twitter:creator` meta tag)
- Add your URL in line 1 of `public/robots.txt`
- Add your links in `src/components/Header.astro`
- Update the intro in `pages/about.md`
- Edit the images in `public/` (optional)
- Edit the RSS styles in `public/` (optional)

After this, you can add your content to `posts/` with Markdown files.

[![Deploy to Netlify](https://www.netlify.com/img/deploy/button.svg)](https://app.netlify.com/start/deploy?repository=https://github.com/cassidoo/blahg)

And finally, please ping me (via social media, or in a GitHub Issue, or whatever) if you use this template! I would love to see your writing and subscribe to your RSS feed!

## Run it yourself

All commands are run from the root of the project, from a terminal:

| Command                          | Action                                                        |
| :------------------------------- | :------------------------------------------------------------ |
| `npm install`                    | Installs dependencies                                         |
| `npm run dev`                    | Starts local dev server at `localhost:4321`                   |
| `npm run build`                  | Build your production site to `./dist/`                       |
| `npm run preview`                | Preview your build locally, before deploying                  |

Note: if you want to edit your RSS feed styles (in `public/rss-styles.xsl`), that does _not_ have hot reloading, so you will have to refresh the page with every change. It seems hard-coded, but that's how XSL styles work (you'll see)!

**Have fun!**
