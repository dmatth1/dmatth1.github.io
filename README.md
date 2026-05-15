# dmatth1.github.io

Personal site at https://dmatth1.github.io/. Hugo + the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.
Built and deployed via GitHub Actions on every push to `main`.

Source was prepared in `evolve/user-site/` and imported here.

## Adding posts

```sh
hugo new content posts/my-new-post.md
# Edit content/posts/my-new-post.md. Set `draft: false` when ready.
git add . && git commit -m "Add post" && git push
```

The Action rebuilds and redeploys automatically.

## Local preview

Requires [Hugo extended](https://gohugo.io/installation/) (≥ 0.128).

```sh
# Theme is fetched by the workflow; for local preview, fetch it once:
git clone --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
hugo server -D    # -D includes drafts; preview at http://127.0.0.1:1313
```

## Customisation

- **`hugo.toml`** — `title`, `params.author`, `params.description`,
  `params.homeInfoParams.Title` / `Content`.
- **`content/about.md`** — bio.
- **`baseURL`** in `hugo.toml` — if pointing a custom domain at the repo,
  set `baseURL` accordingly and add a `static/CNAME` file.
- **Favicon** — drop `favicon.ico` and `favicon-16x16.png` into `static/`
  and update `params.assets.favicon*` in `hugo.toml`.
