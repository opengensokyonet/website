# opengensokyo.net

Source of [opengensokyo.net](https://opengensokyo.net/), the website of Open Gensokyo Network. It is built with [Hugo](https://gohugo.io/) and the [Hextra](https://github.com/imfing/hextra) theme, and is served by Cloudflare Workers static assets.

## Development

Requirements: Hugo 0.168.0 or later and Go 1.24 or later. Go is needed because the theme is a Hugo module pinned in `go.mod`.

From the repository root:

```sh
hugo server
```

The site is then available at <http://localhost:1313/>. To produce the deployable output in `public/`:

```sh
hugo --gc --minify
```

Blog posts go in `content/blog/`, one Markdown file per post with `title` and `date` in the front matter.

## Deployment

Pushes to `main` are built and deployed by Cloudflare Workers Builds with the build command `hugo --gc --minify` and the deploy command `npx wrangler deploy`. The build pins Hugo through the `HUGO_VERSION` build variable. `wrangler.jsonc` binds the output to `opengensokyo.net`.

## License

Text content is licensed under [CC BY 4.0](LICENSE-CC-BY). Code, including code snippets in posts and the site templates, is dual-licensed under the [MIT](LICENSE-MIT) and [Apache 2.0](LICENSE-APACHE) licenses.
