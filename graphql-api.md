# In Context API v4

The In Context API lets you create and edit In Context projects from your own systems, send respondents into a study, and read back the behavioral data that was recorded. It is a single [GraphQL](https://graphql.org/learn/) endpoint.

| | |
| --- | --- |
| Endpoint | `POST https://api.incontext-research.com/v4/graphql` |
| Authentication | `Authorization: Secret <your API secret>` header |
| Format | JSON request body with `query` and optional `variables` and `operationName` |
| Interactive explorer | [portal.incontext-research.com/docs/explorer](https://portal.incontext-research.com/docs/explorer) (requires a Portal login) |
| Source of this page | [github.com/eye-square/docs](https://github.com/eye-square/docs/blob/master/graphql-api.md) |

Related guides: [survey integration](https://incontext-research.com/docs/surveyintegration/), [supported ad formats](https://incontext-research.com/docs/adsupport/), [system requirements](https://incontext-research.com/docs/requirements/), [metrics per format](https://e2.ms/apimetrics), [supported languages per context](https://e2.ms/local), [plugins and webhooks](https://github.com/eye-square/docs/blob/master/plugins/plugins.md).

## Contents

- [Quickstart](#quickstart)
- [Concepts](#concepts)
- [Authentication](#authentication)
- [Making requests](#making-requests)
- [Pagination](#pagination)
- [Errors](#errors)
- [Task data and format discovery](#task-data-and-format-discovery)
- [Project reference](#project-reference)
- [Supported context types](#supported-context-types)
- [Presenter and setup tasks](#presenter-and-setup-tasks)
- [Instructions](#instructions)
- [Deployment filters](#deployment-filters)
- [Sending respondents and survey integration](#sending-respondents-and-survey-integration)
- [Success evaluation](#success-evaluation)
- [Query test results](#query-test-results)
- [Organization settings](#organization-settings)
- [Changelog](#changelog)
- [Support](#support)

---

# Quickstart

This walks you from an API secret to your first results in six steps. Every example on this page can be pasted into the [explorer](https://portal.incontext-research.com/docs/explorer) or sent with `curl`.

## Step 1 Get an API secret

Organization admins create and manage API secrets in the Accounts app: [accounts.incontext-research.com/organizations/me/secrets](https://accounts.incontext-research.com/organizations/me/secrets). A secret belongs either to a team or to the whole organization (see [Team and organization secrets](#team-and-organization-secrets)).

Use the secret only from your server. Never ship it to a browser, an app, or a survey page.

## Step 2 Check who you are

```sh
curl https://api.incontext-research.com/v4/graphql \
  -H 'Content-Type: application/json' \
  -H "Authorization: Secret $INCONTEXT_API_SECRET" \
  -d '{ "query": "{ me { role ... on Team { name } ... on Organization { name } } }" }'
```

```json
{ "data": { "me": { "role": "TEAM", "name": "your-team-name" } } }
```

[Open in explorer](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++me+%7B%0A++++role%0A++++...+on+Team+%7B+name+%7D%0A++++...+on+Organization+%7B+name+%7D%0A++%7D%0A%7D%0A&variables=)

If `role` is `PUBLIC`, the secret was not accepted: check the header name, the `Secret` prefix and the value. An unauthenticated request still succeeds, but the schema then contains only `me`, so a missing field or mutation usually means an authentication problem rather than a missing feature.

## Step 3 Pick an ad format

List the available context types and look at the data a format needs:

```graphql
{
  __type(name: "CONTEXT_TASK_TYPES") {
    enumValues { name description }
  }
}
```

```graphql
{
  __type(name: "ContextTaskDataFacebookNewsfeedVideoAutoplay") {
    fields { name description type { kind name ofType { name } } }
  }
}
```

Ask for a ready-made group template to see the defaults and the supported devices for a format and task sequence:

```graphql
{
  subjectGroupTemplate(contextType: facebookNewsfeedVideoAutoplay, taskSequence: TASK) {
    devices
    description
    template {
      device
      contextMode
      tasks { nodes { type ... on ContextTask { contextType timeout exposureDuration userGuidance } } }
    }
  }
}
```

[Open in explorer](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++subjectGroupTemplate%28contextType%3A+facebookNewsfeedVideoAutoplay%2C+taskSequence%3A+TASK%29+%7B%0A++++devices%0A++++description%0A++++template+%7B%0A++++++device%0A++++++contextMode%0A++++++tasks+%7B+nodes+%7B+type+...+on+ContextTask+%7B+contextType+timeout+exposureDuration+userGuidance+%7D+%7D+%7D%0A++++%7D%0A++%7D%0A%7D%0A&variables=)

## Step 4 Create a project

```graphql
mutation CreateProject($data: ProjectInput!) {
  projectCreate(data: $data) {
    id
    subjectGroups {
      nodes {
        id
        entryUrl
      }
    }
  }
}
```

Variables:

```json
{
  "data": {
    "description": "Red Cross video test - Facebook feed",
    "language": "en_US",
    "billingReference": "PO-12345",
    "subjectGroups": [
      {
        "id": "fb-desktop",
        "device": "desktop",
        "contextMode": "public",
        "tasks": [
          {
            "id": "exposure",
            "type": "context",
            "contextType": "facebookNewsfeedVideoAutoplay",
            "adName": "Red Cross",
            "userGuidance": false,
            "timeout": 120,
            "exposureDuration": 60,
            "data": {
              "brandName": "American Red Cross",
              "text": "Join us today at redcross.org",
              "logoSrc": {
                "imageUrl": "https://d2dom67l5t3ck7.cloudfront.net/demos/redcrosslogoUS.jpg"
              },
              "mediaSrc": {
                "videoUrl": "https://d2dom67l5t3ck7.cloudfront.net/demos/AmericanRedCross15.mp4"
              },
              "brandUrl": "https://www.facebook.com/redcross/"
            }
          }
        ]
      }
    ]
  }
}
```

The response contains the generated project ID (for example `2026-10_AB001`) and one `entryUrl` per subject group. Media URLs must be public HTTPS links: the API downloads them during validation and rejects the request if a file cannot be reached. Always send `subjectGroups`, even as an empty list `[]`.

## Step 5 Send respondents

Link respondents from your survey to the group's `entryUrl`, adding your respondent ID as `UserHash`:

```
https://schedule.incontext-research.com/scheduler/entry?ProjectID=2026-10_AB001&SubjectGroupID=fb-desktop&UserHash=RESPONDENT_ID
```

At the end of the study the respondent is sent back to your survey with `success`, `eligible` and other parameters. See [Sending respondents and survey integration](#sending-respondents-and-survey-integration).

## Step 6 Fetch results

```graphql
query Results($projectId: CustomID!) {
  project(id: $projectId) {
    metrics(first: 50) {
      totalCount
      pageInfo { hasNextPage endCursor }
      nodes {
        subjectGroupId
        taskId
        subjectId
        elements(first: 20) {
          nodes {
            id
            type
            metrics(first: 100) { nodes { name value } }
          }
        }
      }
    }
  }
}
```

```json
{ "projectId": "2026-10_AB001" }
```

`subjectId` is the `UserHash` you passed in step 5, so you can join the results to your survey data. See [Query test results](#query-test-results).

---

# Concepts

A **project** is one study. It contains one or more **subject groups** (often called cells): one test setup that a respondent is routed into. Each subject group has a device class (`desktop` or `mobile`) and an ordered list of **tasks**.

| Task `type` | What the respondent does | Configured with |
| --- | --- | --- |
| `context` | Browses a realistic replica of a platform (Facebook, Instagram, TikTok, YouTube and others) that contains your ad | `contextType` and `data` |
| `presenter` | Watches ads in a fixed presenter, for example a TV player | `presenterType` and `data` |
| `setup` | Answers a short setup step, for example choosing topics of interest | `setupType` and `data` |
| `survey` | Returns to your survey in the middle of the study and comes back afterwards | no further fields |

Tasks can be combined in any order. A typical **task, survey, task** group exposes the respondent to the ad, returns them to your survey for intermediate questions, and then shows a second, guided exposure:

```json
{
  "id": "fb-tst",
  "device": "mobile",
  "contextMode": "public",
  "tasks": [
    { "id": "exposure-1", "type": "context", "contextType": "facebookNewsfeedVideoAutoplay", "userGuidance": false, "...": "..." },
    { "id": "mid-survey", "type": "survey" },
    { "id": "exposure-2", "type": "context", "contextType": "facebookNewsfeedVideoAutoplay", "userGuidance": true, "...": "..." }
  ]
}
```

Key task settings:

| Field | Meaning |
| --- | --- |
| `exposureDuration` | Seconds the respondent can be exposed to the context. The task ends when it runs out. |
| `timeout` | Maximum task duration in seconds. If no exposure duration is set, the task ends after the timeout. |
| `userGuidance` | Guides the respondent to the test ad: newsfeeds scroll to the ad and lock navigation, stories open the ad directly, YouTube ends the task after the pre-roll. |
| `randomization` | Tasks with randomization enabled are shuffled among each other. |
| `adName` | Ad name used in instructions, for example "Please now look for the {{name}} ad". |

`contextMode` is always `public`: a replica that uses publicly available content and needs no browser extension, app or platform login. The value `personal` is no longer supported.

---

# Authentication

Send your API secret in the `Authorization` header with the prefix `Secret`:

```
Authorization: Secret <your API secret>
```

## Roles

`me { role }` tells you how the request was authenticated:

| `role` | Meaning |
| --- | --- |
| `TEAM` | Authenticated with a team secret |
| `ORGANIZATION` | Authenticated with an organization secret |
| `USER`, `ADMIN` | Authenticated as a Portal user (for example in the explorer) |
| `PUBLIC` | Not authenticated. Only `me` is available. |

The schema you see depends on how you are authenticated, so introspect with the same credential your integration uses.

## Team and organization secrets

A team secret reads and creates projects of its team. An organization secret works across the organization's teams: `projects` and `projectCreate` accept optional `organizationId` and `teamId` arguments, and new projects go to the organization's default team when no team is given. Use `organization { id name }` and `organizationTeam { id name }` to look up the IDs.

## Portal login in the explorer

The explorer can send requests either with your Portal login or with an API secret that you paste in. Pick the credential in the dropdown next to the editor and test with the same kind of credential your integration will use.

The older email and password token login (`/v4/auth`) is deprecated. Use API secrets for all integrations.

---

# Making requests

Send a `POST` request with a JSON body to `https://api.incontext-research.com/v4/graphql`:

```json
{
  "query": "query Project($id: CustomID!) { project(id: $id) { id description } }",
  "variables": { "id": "2026-10_AB001" },
  "operationName": "Project"
}
```

The root operations are:

| Operation | Purpose |
| --- | --- |
| `me` | Current identity and role |
| `organization`, `organizationTeam` | Your organization and team |
| `project(id)` | One project, including its groups, tasks, metrics and timelines |
| `projects(first, last, after, before, organizationId, teamId)` | All projects you can access, oldest first |
| `search(type: PROJECT, query, first, ...)` | Text search across your projects |
| `subjectGroupTemplate(contextType, taskSequence)` | Default group setup and supported devices for a format |
| `settings` | Organization defaults, for example success parameters |
| `plugins`, `plugin`, `pluginInstallations`, `pluginInstallation` | Available and installed plugins |
| `projectCreate(data, organizationId, teamId)` | Create a project (mutation) |
| `projectUpdate(id, data)` | Update a project (mutation) |

There are no subscriptions, and the API does not upload media: host images and videos yourself at public HTTPS URLs.

## Updating a project

`projectUpdate` takes a full `ProjectInput`. Read the project first, change what you need, and send back `description`, `language` and the complete `subjectGroups` list: the groups you send replace the existing ones. Read-only fields that appear in query results (for example `entryUrl`, or the `width` and `height` of images) are not accepted as input, so remove them before sending.

---

# Pagination

Lists use the [GraphQL connection model](https://graphql.org/learn/pagination/#complete-connection-model). They take four arguments:

| Argument | Meaning |
| --- | --- |
| `first: Int` | The first n entries, or the first n after the `after` cursor |
| `last: Int` | The last n entries, or the last n before the `before` cursor |
| `after: Cursor` | Start after this cursor |
| `before: Cursor` | End before this cursor |

Rules:

- Always pass `first` or `last` on `projects`, `search`, `metrics` and `timelines`. Without one, the request fails with an internal server error.
- Use either `first` or `last`, not both, and either `after` or `before`, not both.
- The limit must be a positive integer. Keep pages small (for example 50 to 100) and page with cursors instead of requesting everything at once.
- `projects` is ordered oldest first, so `projects(last: 10)` returns your ten newest projects.
- Nested lists of a single project, such as `subjectGroups` and `tasks`, can be read without arguments.

Every connection returns `pageInfo`, `totalCount`, `nodes` and `edges`:

```graphql
{
  projects(last: 2) {
    pageInfo {
      startCursor      # cursor of the first returned entry
      endCursor        # cursor of the last returned entry
      hasNextPage      # more entries after endCursor
      hasPreviousPage  # more entries before startCursor
    }
    totalCount         # number of entries in the whole list
    nodes { id description createdAt }
    edges { cursor node { id } }
  }
}
```

To walk through a long list, repeat the query with `after: <endCursor>` while `hasNextPage` is `true`:

```graphql
query Page($after: Cursor) {
  projects(first: 50, after: $after) {
    pageInfo { hasNextPage endCursor }
    nodes { id description }
  }
}
```

---

# Errors

GraphQL errors are returned with HTTP status `200` in the `errors` array of the response, next to any `data` that could still be resolved. Always check `errors` before using `data`.

Each error has a `type`, a `message` and a nested list of `errors`, each with an `origin`: the path to the field that caused it. A string in the path is a property name and a number is a list index.

```json
{
  "errors": [
    {
      "type": "InvalidDataError",
      "message": "Invalid data",
      "errors": [
        {
          "message": "invalid video url - format has to be .mp4",
          "origin": ["subjectGroups", 0, "tasks", 0, "data", "mediaSrc"]
        },
        {
          "message": "invalid image url - cannot access image under https://example.com/logo.jpg",
          "origin": ["subjectGroups", 0, "tasks", 0, "data", "logoSrc"]
        }
      ]
    }
  ],
  "data": { "projectCreate": null }
}
```

The first origin above points to `data.subjectGroups[0].tasks[0].data.mediaSrc`. All validation problems of a request are reported together.

| `type` | Cause | What to do |
| --- | --- | --- |
| `InvalidDataError` | The project input failed validation, for example an unreachable media URL or a device the format does not support | Fix the fields named in `origin` and resend |
| `GraphQLError` | The query does not match the schema, for example an unknown field or a wrong enum value | Check the query against the schema in the explorer |
| `ClientError` | The request was understood but cannot be carried out, for example missing permissions | Check the credential and the IDs |
| `InternalServerError` | Unexpected server error. The message contains an error ID. | Retry once. If it persists, send the error ID to support. |

A project that does not exist, or that your credential cannot access, is returned as `null` without an error.

A create or update rejected with `InvalidDataError` or `GraphQLError` saves nothing. After a timeout or an internal server error on a mutation, query the project (or `projects(last: 5)`) before retrying, so that you do not create a duplicate.

---

# Task data and format discovery

The `data` field of a task is a JSON object whose shape depends on the `contextType` (or `presenterType` and `setupType`). Because it is JSON, the explorer cannot autocomplete inside it and may show a false "Expected value of type JSON" warning. You can ignore that warning: the server validates `data` and reports real problems in the response.

Each format's data is described by a schema type named `ContextTaskData` plus the format name with a capital first letter, for example `ContextTaskDataFacebookNewsfeedStatic` for `facebookNewsfeedStatic`. The enums `CONTEXT_TASK_DATA_TYPES`, `PRESENTER_TASK_DATA_TYPES` and `SETUP_TASK_DATA_TYPES` list all of them.

```graphql
{
  __type(name: "ContextTaskDataFacebookNewsfeedStatic") {
    fields { name description type { kind name ofType { name } } }
  }
}
```

Media rules that apply to most formats:

- Images: `{ "imageUrl": "https://..." }`, `.jpg` or `.png`.
- Videos: `{ "videoUrl": "https://..." }`, `.mp4`.
- URLs must use HTTPS and be publicly reachable when you create or update the project.
- Query results add read-only fields such as `width`, `height` or `length`. Remove them before reusing a task as input.

Not every format supports both devices. Check `subjectGroupTemplate(...) { devices }` before creating a group: a mismatch is rejected with `given device is not supported for given task`.

---

# Project reference

The tables below summarize the input types. The schema in the explorer holds the complete, current definitions and descriptions.

## ProjectInput

| Field | Type | Notes |
| --- | --- | --- |
| `description` | `String!` | Client, project and contact information. Used for billing. |
| `language` | `LANGUAGES!` | Language of instructions and platform UI, for example `en_US`. See [supported languages](https://e2.ms/local). |
| `billingReference` | `String` | Job, project or PO number for billing |
| `allowReentry` | `Boolean` | Lets respondents reopen an unfinished session with the same entry URL within two hours. They restart the task they left. |
| `deploymentFilters` | `DeploymentFilterInput` | Allowed operating systems and browsers. See [Deployment filters](#deployment-filters). |
| `subjectGroups` | `[SubjectGroupInput]` | The groups of the project. Always send it, as `[]` if empty. |

Project IDs are generated when you create a project. You cannot choose them.

## SubjectGroupInput

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `CustomID` | Group ID, unique within the project. Used in entry URLs and results. |
| `device` | `desktop` or `mobile` | Device class. Mobile covers smartphones and tablets. |
| `contextMode` | `public` | Always `public` |
| `tasks` | `[TaskInput]` | Ordered list of tasks |
| `returnUrl` | `String` | Template for a custom return URL. See [Custom return URL](#custom-return-url). |
| `deploymentFilters` | `DeploymentFilterInput` | Overrides the project's deployment filters for this group |
| `plugins` | `[SubjectGroupPluginSetupInput]` | Plugins to run in this group (`name` and `options`) |

## TaskInput

| Field | Type | Used by | Notes |
| --- | --- | --- | --- |
| `id` | `CustomID` | all | Task ID, unique within the group. Used in results. |
| `type` | `TASK_TYPES` | all | `context`, `presenter`, `setup` or `survey` |
| `contextType` | `CONTEXT_TASK_TYPES` | context | See [Supported context types](#supported-context-types) |
| `presenterType` | `PRESENTER_TASK_TYPES` | presenter | `tvPlayer` or `onlineSlideshow` |
| `setupType` | `SETUP_TASK_TYPES` | setup | `topicSelection` |
| `data` | `JSON` | context, presenter, setup | Format-specific data |
| `exposureDuration` | `Duration` (seconds) | context | Exposure time |
| `timeout` | `Duration` (seconds) | context, presenter, setup | Maximum task duration |
| `userGuidance` | `Boolean` | context | Guide the respondent to the ad |
| `adName` | `String` | context | Ad name used in instructions |
| `randomization` | `Boolean` | context, presenter | Shuffle with other randomized tasks |
| `instructions` | `TaskInstructionInput` | context, presenter, setup | See [Instructions](#instructions) |
| `successParameters` | `ContextTaskSuccessParametersInput` | context, presenter | See [Success evaluation](#success-evaluation) |
| `taskEndButton` | `TaskEndButtonInput` | context, presenter | Button that lets respondents end the task early |
| `eyeTracking` | `TaskEyeTrackingInput` | context, presenter | Webcam eye tracking with a calibration step |
| `emotionTracking` | `TaskEmotionTrackingInput` | context, presenter | Facial emotion tracking |
| `tracking` | `TaskTrackingInput` | context, presenter | Additional timelines to record |

---

# Supported context types

Use these values as `contextType`. Formats marked "control" show the platform without any test ad, for control groups. For X, the API identifiers keep their original `twitter` prefix.

| Platform | `contextType` | Format |
| --- | --- | --- |
| Facebook | `facebookControl` | Facebook control: Facebook context without any ad. |
| Facebook | `facebookMultiple` | Facebook multi ad setup: Injects multiple ads into the Facebook newsfeed. |
| Facebook | `facebookNewsfeedCarousel` | Facebook feed carousel ad: Injects a carousel ad into the Facebook newsfeed. |
| Facebook | `facebookNewsfeedMidroll` | Facebook feed mid roll ad: Injects a content video together with a mid roll ad into the Facebook newsfeed. |
| Facebook | `facebookNewsfeedStatic` | Facebook feed image ad: Injects a static image ad into the Facebook newsfeed. |
| Facebook | `facebookNewsfeedVideoAutoplay` | Facebook feed video ad: Injects a video ad into the Facebook newsfeed. |
| Facebook | `facebookStoriesCarousel` | Facebook stories carousel ad: Injects a carousel (up to three combined ads) ad into Facebook stories. |
| Facebook | `facebookStoriesContent` | Facebook stories content: Injects a content post into Facebook stories. |
| Facebook | `facebookStoriesStatic` | Facebook stories image ad: Injects a static image ad into the Facebook stories. |
| Facebook | `facebookStoriesVideo` | Facebook stories video ad: Injects a video ad into the Facebook stories. |
| Instagram | `instagramControl` | Instagram control: Instagram context without any ad. |
| Instagram | `instagramMultiple` | Instagram multi ad setup: Injects multiple ads into the Instagram newsfeed. |
| Instagram | `instagramNewsfeedCarousel` | Instagram feed carousel ad: Injects a carousel ad into the Instagram newsfeed. |
| Instagram | `instagramNewsfeedStatic` | Instagram feed image ad: Injects a static image into the Instagram newsfeed. |
| Instagram | `instagramNewsfeedVideoAutoplay` | Instagram feed video ad: Injects a video into the Instagram newsfeed. |
| Instagram | `instagramReelsInfeedVideo` | Instagram reels infeed video ad: Injects a video ad into Instagram reels. |
| Instagram | `instagramStoriesCarousel` | Instagram stories carousel ad: Injects a carousel (up to three combined ads) ad into Instagram stories. |
| Instagram | `instagramStoriesContent` | Instagram stories content: Injects a content post into Instagram stories. |
| Instagram | `instagramStoriesStatic` | Instagram stories image ad: Injects a static image ad into the Instagram stories. |
| Instagram | `instagramStoriesVideo` | Instagram stories video ad: Injects a video ad into the Instagram stories. |
| LinkedIn | `linkedinControl` | LinkedIn control: LinkedIn context without any ad. |
| LinkedIn | `linkedinNewsfeedCarousel` | LinkedIn feed carousel ad: Injects a carousel ad into the LinkedIn newsfeed. |
| LinkedIn | `linkedinNewsfeedStatic` | LinkedIn feed image ad: Injects a static image ad into the LinkedIn newsfeed. |
| LinkedIn | `linkedinNewsfeedVideo` | LinkedIn feed video ad: Injects a video ad into the LinkedIn newsfeed. |
| Snapchat | `snapchatCarousel` | Snapchat discovery carousel ad: Injects a carousel (up to three combined ads) ad into Snapchat stories. |
| Snapchat | `snapchatInlineStatic` | Snapchat discovery inline image ad: Injects a static image ad into the currently viewed Snapchat story. |
| Snapchat | `snapchatInlineVideo` | Snapchat discovery inline video ad: Injects a video ad into the currently viewed Snapchat story. |
| Snapchat | `snapchatMultiple` | Snapchat multi ad setup: Injects multiple ads into the Snapchat feed. |
| Snapchat | `snapchatSpotlightInfeedVideo` | Snapchat spotlight infeed video ad: Injects a video ad into the Snapchat spotlight feed. |
| Snapchat | `snapchatStatic` | Snapchat discovery image ad: Injects a static image ad into the Snapchat stories. |
| Snapchat | `snapchatVideo` | Snapchat discovery video ad: Injects a video ad into the Snapchat stories. |
| TikTok | `tiktokBrandTakeoverStatic` | TikTok brand takeover static: Displays an image in fullscreen mode before showing a TikTok feed. |
| TikTok | `tiktokBrandTakeoverVideo` | TikTok brand takeover video: Displays a video in fullscreen mode before showing a TikTok feed. |
| TikTok | `tiktokControl` | TikTok control: TikTok context without any ad. |
| TikTok | `tiktokInfeedVideo` | TikTok infeed video: Injects a video ad into a TikTok feed. |
| TikTok | `tiktokMultiple` | TikTok multi ad setup: Injects multiple ads into the TikTok feed. |
| TikTok | `tiktokTopviewVideo` | TikTok topview video: Injects a video ad at the top of a TikTok feed. |
| X | `twitterControl` | X control: X context without any ad. |
| X | `twitterNewsfeedStatic` | X feed image ad: Injects a static image into the X newsfeed. |
| X | `twitterNewsfeedVideoAutoplay` | X feed video ad: Injects a video into the X newsfeed. |
| YouTube | `youtubeContent` | YouTube content ad: Displays a thumbnail between suggested videos on the homepage, which opens a tracked content video. |
| YouTube | `youtubeControl` | YouTube control: Shows the YouTube context without any ad. |
| YouTube | `youtubeMultiple` | YouTube multi ad setup: Displays multiple ads before YouTube videos. |
| YouTube | `youtubePods` | YouTube pods: Displays a pod before a YouTube video. |
| YouTube | `youtubePreroll` | YouTube preroll ad: Displays a preroll ad before a YouTube video. |
| YouTube | `youtubeShortsContent` | YouTube shorts content ad: Injects a content video into the YouTube shorts feed. |
| YouTube | `youtubeShortsInfeed` | YouTube shorts infeed ad: Injects a video ad into YouTube shorts feed. |
| Youku | `youkuControl` | Youku control: Shows the Youku context without any ad. |
| Youku | `youkuPreroll` | Youku preroll ad: Displays a preroll ad before a Youku video. |
| News site | `newsControl` | News control: Displays a generic news website. |
| News site | `newsMultiple` | News multiple: Displays a generic news website with one or multiple ads. |
| Websites | `websitesControl` | Websites control: Displays a static website without any ads. |
| Websites | `websitesHomepageStatic` | Websites homepage image ad: Injects an image into a static website. |
| Websites | `websitesHomepageVideo` | Websites homepage video ad: Injects a video into a static website. |
| Websites | `websitesMultiple` | Websites multiple: Displays a static website with one or multiple ads. |
| E-commerce | `ecommerceStoreMultiple` | E-commerce store: Injects products into a replica online store. |
| Slides | `slides` | Slides: Sequence of images or videos. |
| Page builder | `builderMultiple` | Page builder: Runs one or more custom pages. |

Format details and examples: [supported ad formats](https://incontext-research.com/docs/adsupport/). The live list is always available through `__type(name: "CONTEXT_TASK_TYPES")`.

---

# Presenter and setup tasks

## TV player

`presenterType: tvPlayer` shows ads in a TV environment. The setup is a list of **blocks** that play in order: content blocks and ad blocks. A content block ends when its first video ends, and an ad block ends after its last ad was seen or skipped. The next block starts right away. Content blocks can offer up to nine channels that respondents switch ("zap") between. Without zapping, only one channel is allowed.

Options (`data.options`): `allowSkip` (skip a whole block with the "s" key or a swipe down), `zappingWithSpacebar` (desktop) and `zappingWithTap` (mobile). See `PresenterTaskDataTvPlayer` in the schema for all fields.

![tv flow](https://portal.incontext-research.com/images/docs/tv-flow.jpg)

TV player behavior is recorded as [timelines](#timelines).

## Online slideshow

`presenterType: onlineSlideshow` is a legacy image and video sequence. For new studies use the `slides` context type, which covers the same use case.

## Topic selection

`type: setup` with `setupType: topicSelection` asks respondents to pick topics they are interested in, for example before a feed. See `SetupTaskDataTopicSelection` in the schema.

---

# Instructions

Context, presenter and setup tasks can show instructions before (`pre`) and after (`post`) the task. Each instruction has two required fields:

- `html`: the instruction text, HTML formatted
- `button`: the label of the continue button

Defaults:

- If `instructions` is not set, default instructions in the project language are used: pre instructions for every task, and post instructions after the last task.
- If `instructions` is an empty object `{}`, no instructions are shown.

Placeholders in pre instructions are replaced at runtime:

| Placeholder | Replaced with |
| --- | --- |
| `{{duration}}` | `exposureDuration` of a context task, or `timeout` of a presenter task |
| `{{name}}` | `adName` of a context task |
| `{{context}}` | The context of a context task |

Custom pre instructions and no post instructions:

```json
"instructions": {
  "pre": {
    "html": "You will now see a Facebook newsfeed.<br/>Please browse it for the next {{duration}} seconds as you normally would.",
    "button": "Continue"
  }
}
```

![custom instructions](https://portal.incontext-research.com/images/docs/fb-newsfeed-instructions.png)

Three tasks with no instructions, custom instructions and default instructions:

```json
[
  { "id": "task-1", "instructions": {} },
  {
    "id": "task-2",
    "instructions": {
      "pre": { "html": "Please browse the feed for {{duration}} seconds.", "button": "Continue" },
      "post": { "html": "Thank you.", "button": "Continue" }
    }
  },
  { "id": "task-3" }
]
```

---

# Deployment filters

By default a study accepts the operating systems and browsers listed in the [system requirements](https://incontext-research.com/docs/requirements/). `deploymentFilters` replaces that list on the project or on a single subject group (the group setting wins).

- `os`: list of `{ "name": ..., "min": ... }`
- `browsers`: list of `{ "name": ..., "min": ... }`
- `min` is optional. Without it, every version is allowed.
- Names are case-insensitive and follow [ua-parser-js](https://github.com/faisalman/ua-parser-js), for example `windows`, `mac os`, `linux`, `chrome`, `firefox`, `edge`.

Windows 10 or newer and any Linux, with any browser:

```json
"deploymentFilters": {
  "os": [
    { "name": "windows", "min": "10" },
    { "name": "linux" }
  ]
}
```

macOS 10.15 or newer with Chrome or Firefox 100 or newer:

```json
"deploymentFilters": {
  "os": [{ "name": "mac os", "min": "10.15" }],
  "browsers": [
    { "name": "chrome", "min": "100" },
    { "name": "firefox", "min": "100" }
  ]
}
```

Respondents who do not match are returned to your survey with `eligible=0`.

---

# Sending respondents and survey integration

## Entry URL

Send respondents to the group's `entryUrl` from the API. It has this form:

```
https://schedule.incontext-research.com/scheduler/entry?ProjectID=<project id>&SubjectGroupID=<group id>
```

| Parameter | Required | Meaning |
| --- | --- | --- |
| `ProjectID` | yes | Project ID |
| `SubjectGroupID` | yes | Group ID |
| `UserHash` | recommended | Your respondent ID. It becomes the `subjectId` in the results. It must be unique per project; a respondent who comes back with a used `UserHash` sees a "session expired" page, unless re-entry applies (see below). Without it, an ID is generated. |
| `origin` | optional | Base URL of the return link. Without it, the HTTP referrer of the entry request is used. |
| `SessionID` | for resuming | Resumes an existing session, for example after an intermediate survey. The other parameters can then be omitted. |
| any other parameter | optional | Passed back unchanged on the return URL |

URL-encode all values.

## Return URL parameters

When the respondent reaches a `survey` task or the end of the study, they are sent back to `origin` (or the referrer) with these query parameters, followed by all extra parameters you passed in:

| Parameter | Values | Meaning |
| --- | --- | --- |
| `success` | `0` or `1` | Whether the respondent met the [success criteria](#success-evaluation). While the study is still in progress (`sessionFinished=0`) it is provisionally `1`. |
| `eligible` | `0` or `1` | Whether the respondent's device and browser met the requirements and the study could run |
| `sessionFinished` | `0` or `1` | `1` when the whole study is finished. `0` on an intermediate return. |
| `SessionID` | string | Session identifier. Store it to resume the session. |
| `previousTask` | task ID | The task before the return |

Use `success` for quality filtering only when `sessionFinished=1`.

## Task survey task flow

A `survey` task returns the respondent to your survey in the middle of the study (`sessionFinished=0`). To continue, send them back to:

```
https://schedule.incontext-research.com/scheduler/entry?SessionID=<SessionID from the return URL>
```

Survey programming details: [survey integration](https://incontext-research.com/docs/surveyintegration/).

## Re-entry

With `allowReentry: true` on the project, a respondent who calls the same entry URL again can continue an unfinished session within two hours of its start. They restart the task they left, and progress within that task is lost. Otherwise, or when the session is finished or older than two hours, they see the "session expired" page.

## Custom return URL

`returnUrl` on a subject group replaces the default return URL with a [Handlebars](https://handlebarsjs.com/guide/) template. Use triple braces so that the URL is not HTML-escaped. Available values include `returnUrl` (the default return URL with all parameters above), `eligible`, `session`, `group` and `task`.

Append a fixed parameter to the default return URL:

```json
"returnUrl": "{{{returnUrl}}}&cell=fb-desktop"
```

If the template cannot be rendered, the default return URL is used with an added `returnUrlError` parameter that describes the problem.

---

# Success evaluation

After a task, the recorded metrics are compared with the task's **success parameters**. The result is returned to your survey as `success`.

## Success parameters

| Field | Type | Passes when |
| --- | --- | --- |
| `visible` | `Int` (ms) | The element was visible for longer than this many milliseconds (`visibility_Duration`). `1` means "was visible at all". |
| `played` | `Int` (ms) | The video played for longer than this many milliseconds (`playback_Duration`) |
| `percentPlayed` | `0` to `100` | The video played for more than this share of the task's main video length |
| `loaded` | `Boolean` | The element was loaded (`loaded_Time` is set) |
| `elements` | list | Separate conditions per tracked element. See below. |
| `matchBehavior` | `ANY` or `ALL` | How several matching elements are combined. See below. |
| `min` | `Int` | Minimum number of matching elements. Default `1`. |

Without `elements`, the conditions apply to every tracked element of the task. With `elements`, each entry is evaluated on its own, using only the fields inside that entry. Put `matchBehavior` and `min` inside each element in that case: top-level fields are ignored when `elements` is set.

Each element entry selects tracked elements with `id` and `type` patterns. Patterns are globs: `*` matches any characters, for example `story-*` or `post-0`. Element IDs and types appear in the [metrics results](#metrics).

`matchBehavior`:

| Value | Passes when |
| --- | --- |
| not set | At least `min` elements match, and every matching element meets the condition |
| `ANY` | At least `min` matching elements meet the condition. Others may fail. |
| `ALL` | Every element the setup expects was tracked, and all of them meet the condition |

If no element matching an entry was recorded, the task does not pass. Empty success parameters (`{}`) switch the success check off. Basic plausibility checks on the recorded data still apply to every task and can mark it unsuccessful.

At least one of several ads watched for more than one second:

```json
"successParameters": {
  "elements": [
    { "id": "*", "played": 1000, "matchBehavior": "ANY", "min": 1 }
  ]
}
```

The first story was visible:

```json
"successParameters": {
  "elements": [{ "id": "story-0", "visible": 1 }]
}
```

## Defaults

When a task has no `successParameters`, the organization default for its format is used, depending on whether user guidance is enabled. Parameters set on a task replace the default completely; they are not merged with it. Your organization's current defaults are returned by `settings { successParameters }`, keyed by format name in kebab case (for example `facebook-newsfeed-static`) with `ug` (guided) and `noUg` (not guided) entries.

Platform defaults:

| `contextType` | With user guidance | Without user guidance |
| --- | --- | --- |
| `facebookMultiple` | `{"loaded":true}` | `{"loaded":true}` |
| `facebookNewsfeedCarousel` | `{"elements":[{"id":"carousel-container","visible":1}]}` | `{"elements":[{"id":"carousel-container","visible":1}]}` |
| `facebookNewsfeedMidroll` | `{"elements":[{"id":"midroll-*-item-0","percentPlayed":90}]}` | `{"elements":[{"id":"post-*","visible":1}]}` |
| `facebookNewsfeedStatic` | `{"visible":1}` | `{"visible":1}` |
| `facebookNewsfeedVideoAutoplay` | `{"percentPlayed":90}` | `{"visible":1}` |
| `facebookStoriesCarousel` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `facebookStoriesContent` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `facebookStoriesStatic` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `facebookStoriesVideo` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `instagramMultiple` | `{"loaded":true}` | `{"loaded":true}` |
| `instagramNewsfeedCarousel` | `{"elements":[{"id":"carousel-container","visible":1}]}` | `{"elements":[{"id":"carousel-container","visible":1}]}` |
| `instagramNewsfeedStatic` | `{"visible":1}` | `{"visible":1}` |
| `instagramNewsfeedVideoAutoplay` | `{"percentPlayed":90}` | `{"visible":1}` |
| `instagramReelsInfeedVideo` | `{"percentPlayed":90}` | `{"visible":1}` |
| `instagramStoriesCarousel` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `instagramStoriesContent` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `instagramStoriesStatic` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `instagramStoriesVideo` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `linkedinNewsfeedCarousel` | `{"elements":[{"id":"carousel-container","visible":1}]}` | `{"elements":[{"id":"carousel-container","visible":1}]}` |
| `linkedinNewsfeedStatic` | `{"visible":1}` | `{"visible":1}` |
| `linkedinNewsfeedVideo` | `{"percentPlayed":90}` | `{"visible":1}` |
| `newsMultiple` | `{"loaded":true}` | `{"loaded":true}` |
| `snapchatCarousel` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `snapchatInlineStatic` | `{"visible":1}` | `{"visible":1}` |
| `snapchatInlineVideo` | `{"visible":1}` | `{"visible":1}` |
| `snapchatMultiple` | `{"loaded":true}` | `{"loaded":true}` |
| `snapchatSpotlightInfeedVideo` | `{"percentPlayed":90}` | `{"visible":1}` |
| `snapchatStatic` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `snapchatVideo` | `{"elements":[{"id":"story-0","visible":1}]}` | `{"elements":[{"id":"story-0","visible":1}]}` |
| `tiktokBrandTakeoverStatic` | `{"elements":[{"id":"tiktok-post","visible":1}]}` | `{"elements":[{"id":"tiktok-post","visible":1}]}` |
| `tiktokBrandTakeoverVideo` | `{"elements":[{"id":"tiktok-post","percentPlayed":90}]}` | `{"elements":[{"id":"tiktok-post","visible":1}]}` |
| `tiktokInfeedVideo` | `{"elements":[{"id":"tiktok-post","percentPlayed":90}]}` | `{"elements":[{"id":"tiktok-post","visible":1}]}` |
| `tiktokMultiple` | `{"loaded":true}` | `{"loaded":true}` |
| `tiktokTopviewVideo` | `{"elements":[{"id":"tiktok-post","percentPlayed":90}]}` | `{"elements":[{"id":"tiktok-post","visible":1}]}` |
| `twitterNewsfeedStatic` | `{"visible":1}` | `{"visible":1}` |
| `twitterNewsfeedVideoAutoplay` | `{"percentPlayed":90}` | `{"visible":1}` |
| `websitesHomepageStatic` | `{"visible":1}` | `{"visible":1}` |
| `websitesHomepageVideo` | `{"visible":1}` | `{"visible":1}` |
| `websitesMultiple` | `{"loaded":true}` | `{"loaded":true}` |
| `youkuPreroll` | `{"visible":1}` | `{"visible":1}` |
| `youtubeContent` | `{"loaded":true}` | `{"loaded":true}` |
| `youtubeMultiple` | `{"loaded":true}` | `{"loaded":true}` |
| `youtubePods` | `{"played":5000}` | `{"visible":1}` |
| `youtubePreroll` | `{"played":5000}` | `{"visible":1}` |
| `youtubeShortsContent` | `{"percentPlayed":90}` | `{"visible":1}` |
| `youtubeShortsInfeed` | `{"percentPlayed":90}` | `{"visible":1}` |

Control formats, `slides`, `builderMultiple`, `ecommerceStoreMultiple` and the presenter types have no default success parameters, so only the plausibility checks apply unless you set your own.

---

# Query test results

Metrics and timelines are fields of a project. Both accept the same filters, which can be combined:

| Argument | Meaning |
| --- | --- |
| `subjectGroup: CustomID` | Only this group |
| `task: CustomID` | Only this task ID. Task IDs can repeat across groups, so combine it with `subjectGroup`. |
| `subject: String` | Only this respondent (`UserHash`) |

```graphql
{
  project(id: "2026-10_AB001") {
    metrics(first: 10, subjectGroup: "fb-desktop", task: "exposure", subject: "RESPONDENT_ID") {
      nodes { subjectGroupId taskId subjectId }
    }
    timelines(first: 10, subjectGroup: "fb-desktop", task: "exposure", subject: "RESPONDENT_ID") {
      nodes { subjectGroupId taskId subjectId }
    }
  }
}
```

## Metrics

A metric set holds the metrics of one respondent in one task. It contains **elements**: the tracked parts of the setup, usually the test ad and its parts (for example a carousel container and its items). Each element has an `id`, a `type` and a list of metrics with `name`, `value` and `timestamp`.

```json
{
  "subjectGroupId": "fb-desktop",
  "taskId": "exposure",
  "subjectId": "RESPONDENT_ID",
  "elements": {
    "nodes": [
      {
        "id": "carousel-container",
        "type": "facebook-post-carousel",
        "metrics": {
          "nodes": [
            { "name": "loaded_Time", "value": "1326", "timestamp": "1791993361701" },
            { "name": "visibility_Duration", "value": "7001", "timestamp": "1791993363872" },
            { "name": "actionUrl_ClickTime", "value": null, "timestamp": null }
          ]
        }
      }
    ]
  }
}
```

- `value` is always a string, or `null` if the event did not happen. Convert it to a number yourself.
- `*_Duration` values are durations and `*_Time` values are points in time since the start of the recording, both in milliseconds. `timestamp` is the Unix time in milliseconds when the metric was first recorded.
- Every element type returns a fixed set of metric names. Metrics that were not recorded are `null`.
- The metrics available per format are listed in the [API metrics overview](https://e2.ms/apimetrics).

## Timelines

A timeline is a sequence of timestamped events of one respondent in one task, for example scroll positions, visible areas or interactions. The available timelines are the fields of `TimelineList` in the schema (for example `StimulusSequence`, `StimulusSequenceHost`, `Interact`, `Scroll`, `VisibleRect`, `CustomEvent`). Each one is a paginated list. Request only the timelines and fields you need, and use small pages.

```graphql
{
  project(id: "2026-10_AB001") {
    timelines(first: 10) {
      nodes {
        subjectGroupId
        taskId
        subjectId
        timelines {
          StimulusSequenceHost(first: 10) { nodes { Timestamp StimulusID } }
          Interact(first: 100) { nodes { Timestamp Value EventType } }
        }
      }
    }
  }
}
```

TV player tasks record these timelines:

| Timeline | Events |
| --- | --- |
| `StimulusSequence` | A video started playing visibly (`StimulusID` is the video ID) |
| `StimulusSequenceHost` | Task start and end (`StimulusID` is `TaskStart` or `TaskEnd`) |
| `Interact` | The respondent switched channels (`EventType: zapping`, `Value` is the video that was playing) |
| `CustomEvent` | Playback stopped, failed to restart or resumed, for example on a bad connection (`Name` is the video, `Value` is the event) |

```json
"StimulusSequence": {
  "nodes": [
    { "Timestamp": "1542967291284", "StimulusID": "content1_ch1" },
    { "Timestamp": "1542967356656", "StimulusID": "spot_1" }
  ]
},
"Interact": {
  "nodes": [
    { "Timestamp": "1542967314343", "Value": "content1_ch1", "EventType": "zapping" }
  ]
}
```

---

# Organization settings

`settings` returns your organization's defaults:

```graphql
{
  settings {
    allowReentry          # default for new projects
    successParameters     # default success parameters per format
    watermark { mode }    # default watermark
  }
}
```

---

# Changelog

| Date | Change |
| --- | --- |
| 2026-10 | Page rewritten: quickstart, current roles and error format, return URL parameters, success parameter rules and current defaults, complete list of context types. Removed the deprecated token login. |
| 2026-07 | API secrets managed in the Accounts app |

---

# Support

For technical questions about this API, email [api-support@eye-square.com](mailto:api-support@eye-square.com). Include the request, the full response and any error ID.
