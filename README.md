<h1 align="center">
  <img src="./images/logo.png" alt="in-context" width="75" height="75">
</h1>

# In Context documentation

Public documentation for the In Context API and plugins.

| File | Description |
| --- | --- |
| [graphql-api.md](./graphql-api.md) | Reference for the In Context GraphQL API (v4): authentication with API secrets, a quickstart, creating projects, sending respondents from a survey, success evaluation, and reading results and metrics. |
| [plugins/plugins.md](./plugins/plugins.md) | Overview of plugins and what they can use: client scripts, webhooks, options and client capabilities. |
| [plugins/client-scripts-api.md](./plugins/client-scripts-api.md) | How to run your own scripts inside a task: lifecycle hooks, jobs, the events a task emits, and tracking your own elements. |
| [plugins/script-development.md](./plugins/script-development.md) | How to serve a script from a local server (http-server or Vite) while you develop it. |
| [plugins/webhooks.md](./plugins/webhooks.md) | How to receive In Context events on your own endpoint, validate their signature, and what each event payload contains. |

`graphql-api.md` is also shown on the **API v4** page of the Portal, [portal.incontext-research.com/docs/api-v4](https://portal.incontext-research.com/docs/api-v4) (Portal login required), which loads it from `master`. A merge to `master` is live there within about five minutes.
