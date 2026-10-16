<h1 align="center">
  <img src="./images/logo.png" alt="in-context" width="75" height="75">
</h1>

# In Context documentation

Public documentation for the In Context API and plugins. This repository is public, so anyone can read these files on GitHub.

## What is in here and where it shows up

| File | Where people read it |
| --- | --- |
| [graphql-api.md](./graphql-api.md) | The **API v4** tab of the Portal, [portal.incontext-research.com/docs/api-v4](https://portal.incontext-research.com/docs/api-v4) (Portal login required), and here on GitHub |
| [plugins/](./plugins/plugins.md) (plugins, client scripts API, script development, webhooks) | Only here on GitHub. The API page links to it. Not published anywhere else that we know of. |
| `images/logo.png` | The logo at the top of this README and the plugin pages |

The Portal's **Main Documentation** tab (`/docs`) does not come from this repository. It is bundled in the Portal code in `eye-square/in-context-testing`.

## How changes go live

- The Portal loads `graphql-api.md` from the `master` branch of this repository every time someone opens the API v4 page. There is no build or deploy step.
- So a merge to `master` is live for all customers within about five minutes (GitHub caches raw files for up to 5 minutes). Review changes carefully before merging.
- Changes to the Portal around the page (header, navigation, table of contents, styling) and new images live in `eye-square/in-context-testing` and need a Portal release.

## Contributing

1. Create a branch from `master`.
2. Edit the Markdown and open a pull request.
3. Get a review, then merge.

Things to keep in mind:

- **Rendering.** The Portal renders GitHub-flavored Markdown with `react-markdown` and `remark-gfm`. The GitHub preview is close, but not identical. For example, the Portal table of contents only lists `#` and `##` headings.
- **Links and images.** The Portal shows the page under its own address, so relative file links and image paths (such as `./plugins/plugins.md`) do not work there. Use absolute URLs; `#heading` links are fine. Images belong in the Portal (`client/apps/portal/public/images/docs/` in `eye-square/in-context-testing`) and are linked as `https://portal.incontext-research.com/images/docs/<file>`. Links to `https://portal.incontext-research.com` are rewritten to the matching staging or development Portal automatically.
- **No secrets or customer data.** Use placeholders such as `$INCONTEXT_API_SECRET`, and made-up project names and respondent IDs.
- **Example IDs** use the current year and month, for example `2026-10_AB001`.
- **Changelog.** Add a line to the **Changelog** section of `graphql-api.md` for changes customers should notice.
