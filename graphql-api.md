# Table of content

- [How to use this api](#how-to-use-this-api)
  - [Graphql](#graphql)
  - [The Graphiql explorer](#the-graphiql-explorer)
  - [Authentication and authorization](#authentication-and-authorization)
  - [Pagination](#pagination)
  - [Non standard JSON-type](#non-standard-json-type)
  - [Error responses](#error-responses)
- [Project model](#project-model)
  - [Context tasks](#context-tasks)
  - [Survey tasks](#survey-tasks)
  - [Presenter tasks](#presenter-tasks)
- [TV player](#tv-player)
- [System entry policies](#system-entry-policies)
- [Session validation](#session-validation)
- [Success evaluation](#success-evaluation)
- [Instructions](#instructions)
- [List of supported languages](#list-of-supported-languages)
- [List of supported operating systems and browsers](#list-of-supported-operating-systems-and-browsers)
- [Query test results](#query-test-results)
  - [Query filters](#query-filters)
  - [Metrics](#metrics)
  - [Timelines](#timelines)
- [Contact](#contact)

# How to use this api

## Graphql

Version 4 of the eye-square api is a complete rebuild based on [graphql](https://graphql.org/). With this documentation comes a full api reference and an interactive explorer to make api calls.

The eyesquare api allows you to easily integrate the eye-square in-context solution into your services.

You can find the graphql endpoint under:

POST [https://api.incontext-research.com/v4/graphql](https://api.incontext-research.com/v4/graphql)

## The Graphiql explorer

Best starting point for using the api is the [interactive explorer](https://portal.incontext-research.com/docs/explorer). Through it you can easily make api calls and have an interactive feedback for doing so.

We also provide you with some example queries that you can choose in the dropdown from the explorer.

Unfortunately, due to some restrictions in the current graphiql explorer you might see some linting errors where there in reality aren't. If you come across "Expected Value of Type JSON" error as shown in the task.data field cou can safely ignore it and make the api call. If something is wrong, the error in the response will tell you what and where.

![falsy error](https://portal.incontext-research.com/images/docs/falsy-explorer-error.png)

## Authentication and authorization

The api supports two different authentication types:

- token based authentication. Use this for client based api calls.
- api secret based authentication. Use this for server to server based calls. Never expose your secret to the client side!

Our interactive explorer is able to use both authentication methods in behalf of your account. Just choose which method you want to use in the credentials dropdown. Team credentials uses the api secret of your team, user credentials use the token of your user account.

As we expose different graphql schemas for different authentication methods (a team might have different access restrictions than a normal user) please make sure to use the correct credentials here when testing queries.

---

### Token based authentication

We use [jwt](https://jwt.io/introduction/)(json web tokens) to authenticate and authorize users. To make an api call, you have to first retrieve a token and then include this token in you api calls. A jwt has an expiration time of 30 days, means after 30 days a user has to request a new token in order to make api calls.

To get a token you have to make a post request with email and password of the user to:

POST [https://api.incontext-research.com/v4/auth](https://api.incontext-research.com/v4/auth)

When authentication of the user was successful you will recieve a user profile including a freshly generated token.

Example request using curl

```sh
curl \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{ "email": "users@email", "password": "superSecretPassword" }' \
  https://api.incontext-research.com/v4/auth
```

Recieved response:

```json
{
  "role": "user",
  "name": "username",
  "email": "user@email",
  "createdAt": {},
  "updatedAt": {},
  "team": {
    "name": "default"
  },
  "organization": {
    "name": "eye-square",
    "code": "ES"
  },
  "token": "your.access.token"
}
```

With the recieved token you can now make requests against our api (the explorer uses the token of the logged in user automatically, so you don't have to worry about this when exploring our api). To do so, set a header 'Authorization' with the value 'Bearer your.access.token'.

Example token request ([view in explorer](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++me+%7B%0A++++...+on+User+%7B%0A++++++name%0A++++%7D%0A++%7D%0A%7D%0A&variables=)):
```sh
curl \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{ "query": "{ me { ...on User { name } }}" }' \
  -H 'Authorization: Bearer your.access.token' \
  https://api.incontext-research.com/v4/graphql
```
Results in:

```json
{
  "data": {
    "me": {
      "name": "username"
    }
  }
}
```

---

### Apisecret based authentication

Each team of an organization has an autogenerated api secret. This secret can be used to make server to server api calls in behalf of the team. You can get the team secret through the explorer:

[explorer link](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++organizationTeam+%7B%0A++++name%0A++++apiSecret%0A++%7D%0A%7D%0A&variables=)

To make an api secret based call, just either

- include the secret as a query param:

  `https://api.incontext-research.com/v4/graphql?apiSecret=yourSecret`

- or set it as a header

  `Authorization: Secret your.team.secret`

Example call from above:
```sh
curl \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{ "query": "{ me { ...on Team { name } }}" }' \
  https://api.incontext-research.com/v4/graphql?apiSecret=yourSecret
```

Results in:

```jsonc
{
  "data": {
    "me": {
      "name": "default" // this is now your teams name
    }
  }
}
```

If you are not sure if you are logged in as user, team or use the public api, you can ask for the role of the authenticated user with the following query:

[`{ "query": "{ me { role }}" }`](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++me+%7B%0A++++role%0A++%7D%0A%7D%0A&variables=)

Roles can be one of

- user
- admin (not supported yet)
- team
- superUser

## Pagination

Every list of elements in the api requires pagination parameters. We are following best practices that can be seen [here](https://graphql.org/learn/pagination/#complete-connection-model).

There are four input params that control pagination:

- first: integer => returns the first x of entries, if after is set, it returns the first x elements after the given Cursor
- last: integer => returns the last x of entries, if before is set, it returns the last x elements before the given Cursor
- after: Cursor
- before: Cursor

Note: first and last as well as after and before exclude each other. Means, that if you have a first param, you cannot set last at the same time.

At the same time the response Object contains a lot of useful information for pagination through the list, it's best explained with an actual query ([view in explorer](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++projects%28last%3A+2%29+%7B+%23+get+the+last+2+projects%0A++++pageInfo+%7B%0A++++++startCursor+%23+cursor+of+first+recieved+edge%0A++++++endCursor+%23+cursor+of+last+recieved+edge%0A++++++hasNextPage+%23+true+if+there+is+content+after+the+last+recieved+edge+%0A++++++hasPreviousPage+%23+true+if+there+is+content+before+the+first+recieved+edge%0A++++%7D%0A++++totalCount+%23+total+number+of+objects+you+can+recieve%0A++++nodes+%7B+%23+shortcut+to+nodes+in+case+you+don%27t+need+a+pagination+cursor%0A++++++...project%0A++++%7D%0A++++edges+%7B%0A++++++node+%7B+%23+the+content+itself%0A++++++++...project%0A++++++%7D%0A++++++cursor+%23+cursor+of+the+edge%2C+only+used+in+pagination+params%0A++++%7D%0A++%7D%0A%7D%0A%0Afragment+project+on+Project+%7B%0A++id%0A%7D&variables=)):

```graphql
{
  projects(last: 2) { # get the last 2 projects
    pageInfo {
      startCursor # cursor of first recieved edge
      endCursor # cursor of last recieved edge
      hasNextPage # true if there is content after the last recieved edge
      hasPreviousPage # true if there is content before the first recieved edge
    }
    totalCount # total number of objects you can recieve
    nodes { # shortcut to nodes in case you don't need a pagination cursor
      ...project
    }
    edges {
      node { # the content itself
        ...project
      }
      cursor # cursor of the edge, only used in pagination params
    }
  }
}

fragment project on Project {
  id
}
```

In a lot of cases you wont need the full capacities of pagination. The power of graphQL lets you decide which functionality you use.

For example if you want only retrieve the last 10 projects ([view in explorer](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++projects%28last%3A+10%29+%7B%0A++++nodes+%7B%0A++++++...project%0A++++%7D%0A++%7D%0A%7D%0A%0Afragment+project+on+Project+%7B%0A++id%0A++description%0A%7D%0A&variables=)):

```graphql
{
  projects(last: 10) {
    nodes {
      ...project
    }
  }
}

fragment project on Project {
  id
  description
}
```

## Non standard JSON-type

Some of our data structures are very complex and would lead to very big Union types/Queries. For this reason, some of our fields are using a custom implementation of JSON type. A Json type can contain anything. This is mostly used in our project.subjectGroups.tasks.data as the data varies based on the Context type. This has the advantage that we keep your queries short, but the downside that we cannot provide autocompletion for fields inside those JSON formats in the explorer.

Example ([view in explorer](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++projects%28first%3A+1%29+%7B%0A++++nodes+%7B%0A++++++subjectGroups+%7B%0A++++++++nodes+%7B%0A++++++++++tasks+%7B%0A++++++++++++nodes+%7B%0A++++++++++++++...+on+ContextTask+%7B%0A++++++++++++++++data+%23+this+is+our+custom+JSON+type+that+can+contain+pretty+much+anything%0A++++++++++++++%7D%0A++++++++++++%7D%0A++++++++++%7D%0A++++++++%7D%0A++++++%7D%0A++++%7D%0A++%7D%0A%7D%0A&variables=)):

```graphql
{
  projects(first: 1) {
    nodes {
      subjectGroups {
        nodes {
          tasks {
            nodes {
              ... on ContextTask {
                data # this is our custom JSON type that can contain pretty much anything
              }
            }
          }
        }
      }
    }
  }
}
```

For all those types we export fitting graphql types in our schema though, so you know which contextType requires which fields ([explorer](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++__type%28name%3A+%22ContextTaskDataFacebookNewsfeedStatic%22%29+%7B%0A++++fields+%7B%0A++++++name%0A++++++description%0A++++%7D%0A++%7D%0A%7D%0A&variables=)):

```graphql
{
  __type(name: "ContextTaskDataFacebookNewsfeedStatic") {
    fields {
      name
      description
    }
  }
}
```

## Error responses

The project model is very versatile and it's easy to make mistakes. We try to validate every possible value/combination to prevent the creation of wrongly configured projects. Additionally we try to point you to the right places if an error occurs.

To do so, we return a custom Error format. It contains the original Error Message, some default graphql fields (locations and path) as well as an origin.

The origin contains the route to the error in form of an array. If the entry is a string, it's an object property, if it's an integer it's a field index of a list.

An Example:

```json
{
  "message": "invalid data at subjectGroups.0.tasks.0.data.mediaSrc - invalid video url",
  "origin": ["subjectGroups", 0, "tasks", 0, "data", "mediaSrc"]
}
```

Translated into a javascript path this would look like:

`project.subjectGroups[0].tasks[0].data.mediaSrc`

Meaning, the mediaSrc of the specified task is invalid.

# Project model

The most important parts of the eye-square api are projects. A project is a set of test setups. A test setup is called a Subject Group. Each Subject Group contains a list of routines called Tasks. Each task presents a different part of a setup. There are different kind of tasks:

- ContextTask: a subject is shown an ad inside of a context (for example facebook) and behaviour is measured.
- SurveyTask: an external survey is shown, usually only needed between other tasks.
- PresenterTask: a subject is shown different ads in a choosen presenter (TV, Online Slideshow, ...)

Tasks can be combined in any order. For example:

```jsonc
 "project": {
   "id": "2018-01_ES001",
   "subjectGroups": [{
     "id": "first group",
     "tasks": [
       {
         "type": "context",
         "contextType": "facebookNewsfeedVideoAutoplay"
         // ...
       },
       {
         "type": "survey"
       },
       {
         type: "presenter",
         persenterType: "tvPlayer"
         // ...
       }
     ],
     // ...
   }],
   // ...
 }
```

This project contains only one subject group. This group contains three tasks, one for each type.

When a subject would enter this test setup it would first get shown a video ad on facebook. Afterwards it would get send to an external survey. When done with the survey the last task would show a TV-setup.

![task flow](https://portal.incontext-research.com/images/docs/task-flow.jpg)

You find a lot of example project setups in the Explorer, just choose one in the Examples dropdown.

## Context tasks

A context task shows an ad in a defined context (Facebook for example) and measures behaviour. A context tasks has different phases, most of them optional:

- pre instructions (optional, see further below): some instructions that are shown before the task really starts
- context browsing: the subject is directed to the choosen context and is shown an ad, only required step of a context task.
- post instructions (optional, see further below): instructions a subject sees after the context task.

![context task flow](https://portal.incontext-research.com/images/docs/context-task-flow.jpg)

### Supported context types

The following types are currently API supported:

[view in explorer](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++__type%28name%3A+%22CONTEXT_TASK_TYPES%22%29+%7B%0A++++enumValues+%7B%0A++++++name%0A++++++description%0A++++%7D%0A++%7D%0A%7D%0A&variables=)

Additional information about the ad formats can be found here: [http://e2.ms/adsupport](http://e2.ms/adsupport)

Each Context type has different setup params. If you want to know which data is required for setup you can ask our graphql schema by searching for "ContextTaskData<Ad-Type>" Where ad type is one of the types you retrieved above, for example [ContextTaskDataFacebookNewsfeedVideoAutoplay](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++__type%28name%3A+%22ContextTaskDataFacebookNewsfeedVideoAutoplay%22%29+%7B%0A++++name%0A++++description%0A++++fields+%7B%0A++++++name%0A++++++description%0A++++%7D%0A++%7D%0A%7D%0A&variables=)

## Survey tasks

As for now survey tasks don't require any additional setup and we send the subjects back to the provided referrer when entering our system.

The system allows to re-enter the survey between exposures. An intermission call will be made to either the HTTP referrer or the origin if no referrer is provided. The system appends a SessionID that must be returned back for follow up exposures:

BaseUrl: https://schedule.incontext-research.com/scheduler/entry/

| Name      | Type    | Description                                                                                               |
| --------- | ------- | --------------------------------------------------------------------------------------------------------- |
| sessionID | integer | A session identifier we send you in the intermission step. Please save this value and pass it back to us. |

## Presenter tasks

A presenter task has the same phases as a context task. But instead of showing an ad inside a context environment the subject is presented an ad setup in a fixed environment.

### TV player

The TV player lets you setup up to 9 "TV channels" with different content and ads that are shown in between.
The core unit of the setup is a block. You can setup as many blocks as you want and blocks are shown in consecutive order.

You can alternate/setup as many blocks as you want. A content block ends when the first video ends. An ad block ends after the last ad was seen/skipped. After a block ends, the next block is shown right away.

![tv flow](https://portal.incontext-research.com/images/docs/tv-flow.jpg)

---

**RECORDED DATA**

Respondent behavior in TV tasks is recorded as timelines of events with timestamps. There is an example query setup in the explorer.

The following events are distinguished:

_StimulusSequence_

Gives back events when a video file started visibly playing.

```json
"StimulusSequence": {
  "nodes": [
    {
      "Timestamp": "1542967291284",
      "StimulusID": "content1_ch1"
    },
    {
      "Timestamp": "1542967356656",
      "StimulusID": "spot_1"
    },
    {
      "Timestamp": "1542967386612",
      "StimulusID": "content2_ch1"
    }

  ]
}
```

_StimulusSequenceHost_

Gives back absolute task start and end.

```json
"StimulusSequenceHost": {
  "nodes": [
    {
      "Timestamp": "1542967291279",
      "StimulusID": "TaskStart"
    },
    {
      "Timestamp": "1542967447604",
      "StimulusID": "TaskEnd"
    }
  ]
}
```

_Interact_

Gives back when the user zapped away from a video. "Value" is the video ID that was playing when the user zapped away.

```json
"Interact": {
  "nodes": [
    {
      "Timestamp": "1542967314343",
      "Value": "content1_ch1",
      "EventType": "zapping"
    },
    {
      "Timestamp": "1542967369191",
      "Value": "spot_1",
      "EventType": "zapping"
    }
  ]
}
```

_CustomEvents_

Gives back events if the playback of any video was stopped for whatever reason (e.g. bad internet connection).

```json
"CustomEvent": {
  "nodes": [
    {
      "Timestamp": "1542889006504",
      "Name": "ch1_1",
      "Value": "stopped"
    },
    {
      "Timestamp": "1542889006505",
      "Name": "ch1_1",
      "Value": "restart failed"
    },
    {
    "Timestamp": "1542889170304",
    "Name": "ch1_1",
    "Value": "resumed"
    }
  ]
}
```

# System entry policies

The system entry to start and resume a session is  
[https://schedule.incontext-research.com/scheduler/entry](https://schedule.incontext-research.com/scheduler/entry)

A session start requires at least the parameters ProjectID and SubjectGroupID. The UserHash is optional but recommended. If no UserHash is provided, the system will provide a unique user hash. If a UserHash is provided and if it is unique for the called project, stimulus presentation and data collection will be performed normally. If a UserHash already exists in the system, the user will see a termination page:
_"Your session is expired. Unfortunately, you can only access this survey once. If you have questions or concerns, please contact us at [app-support@eye-square.com](mailto:app-support@eye-square.com) and include your SessionID: 123456"_

A session resume for the case "task-survey-task" (see chapter Survey Intermission) has to be directed to the same URL:

[https://schedule.incontext-research.com/scheduler/entry?SessionID=](https://schedule.incontext-research.com/scheduler/entry?SessionID=)

but requires the SessionID as an additional parameter (all other parameters can be omitted).

# Session validation

The following parameters will be sent back to the survey to be used for filtering/disqualifying respondents:

| Name     | Type           | Description                                                                                                                                                                              |
| -------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| payload  | GET parameters | All variables the system gets passed into will also be returned unmodified.                                                                                                              |
| eligible | integer        | Will be passed after every exposure. Defines if the respondent meets the browser/OS requirements and has successfully installed the browser extension (if required). Values are 0 and 1. |
| success  | integer        | Will be passed after every exposure. Defines if the respondent completed the task. Criteria for success depends on the project configuration. Values are 0 and 1.                        |

# Success evaluation

The success of a task can be defined and evaluated on a per project basis. Based on those definitions we analyze the validity of collected metrics. Different evaluation rules are applied by default based on the ad formats and if a task has user guidance enabled or not. These evaluation rules can be customized for each task.

If no metrics are recorded we don't evaluate success and return always 1.

## Context tasks defaults

In the table below you see the defaults used:

|Context Type|Guided Tasks|Non Guided Tasks|
|----------|----------|----------|
|facebook-multiple|{"loaded":true}|{"loaded":true}|
|facebook-newsfeed-static|{"visible":1}|{"visible":1}|
|facebook-newsfeed-video-autoplay|{"percentPlayed":90}|{"visible":1}|
|facebook-newsfeed-carousel|{"elements":[{"id":"carousel-container","visible":1}]}|{"elements":[{"id":"carousel-container","visible":1}]}|
|facebook-newsfeed-midroll|{"elements":[{"id":"post-*-midroll-*-item-0","percentPlayed":90}]}|{"elements":[{"id":"post-*","visible":1}]}|
|facebook-newsfeed-content-video|{"percentPlayed":90}|{"visible":1}|
|facebook-stories-static|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|facebook-stories-video|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|facebook-stories-carousel|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|facebook-stories-content|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|instagram-multiple|{"loaded":true}|{"loaded":true}|
|instagram-newsfeed-static|{"visible":1}|{"visible":1}|
|instagram-newsfeed-video-autoplay|{"percentPlayed":90}|{"visible":1}|
|instagram-newsfeed-carousel|{"elements":[{"id":"carousel-container","visible":1}]}|{"elements":[{"id":"carousel-container","visible":1}]}|
|instagram-reels-infeed-video|{"percentPlayed":90}|{"visible":1}|
|instagram-stories-static|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|instagram-stories-video|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|instagram-stories-carousel|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|instagram-stories-content|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|snapchat-static|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|snapchat-video|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|snapchat-carousel|{"elements":[{"id":"story-0","visible":1}]}|{"elements":[{"id":"story-0","visible":1}]}|
|snapchat-inline-static|{"visible":1}|{"visible":1}|
|snapchat-inline-video|{"visible":1}|{"visible":1}|
|tiktok-multiple|{"loaded":true}|{"loaded":true}|
|tiktok-brand-takeover-static|{"elements":[{"id":"tiktok-post","visible":1}]}|{"elements":[{"id":"tiktok-post","visible":1}]}|
|tiktok-brand-takeover-video|{"elements":[{"id":"tiktok-post","percentPlayed":90}]}|{"elements":[{"id":"tiktok-post","visible":1}]}|
|tiktok-infeed-video|{"elements":[{"id":"tiktok-post","percentPlayed":90}]}|{"elements":[{"id":"tiktok-post","visible":1}]}|
|tiktok-topview-video|{"elements":[{"id":"tiktok-post","percentPlayed":90}]}|{"elements":[{"id":"tiktok-post","visible":1}]}|
|twitch-preroll|{"played":5000}|{"visible":1}|
|twitch-midroll|{"played":5000}|{"visible":1}|
|twitter-newsfeed-static|{"visible":1}|{"visible":1}|
|twitter-newsfeed-video-autoplay|{"percentPlayed":90}|{"visible":1}|
|websites-homepage-static|{"visible":1}|{"visible":1}|
|websites-homepage-video|{"visible":1}|{"visible":1}|
|websites-multiple|{"loaded":true}|{"loaded":true}|
|youku-preroll|{"visible":1}|{"visible":1}|
|youtube-discovery|{"loaded":true}|{"loaded":true}|
|youtube-content|{"loaded":true}|{"loaded":true}|
|youtube-frequency|{"played":5000}|{"visible":1}|
|youtube-pods|{"played":5000}|{"visible":1}|
|youtube-preroll|{"played":5000}|{"visible":1}|
|youtube-masthead|{"elements":[{"id":"youtube-content-video","played":5000}]}|{"elements":[{"id":"mastheadAd-0","visible":1}]}|
|youtube-multiple|{"loaded":true}|{"loaded":true}|
|youtube-shorts-infeed|{"percentPlayed":90}|{"visible":1}|
|youtube-shorts-content|{"percentPlayed":90}|{"visible":1}|

# Instructions

Presenter and Context tasks can have instructions shown before (pre instructions) and after the task (post instructions).

A instruction is made of following fields:

- html: html that is shown as the instruction text.
- button: content of the instruction button.

Special cases:

- if no instructions (`null/undefined`) are set, default instructions are used.
- if instructions is an empty onject (`{}`), no instructions are shown.

We will add default instructions in the following way:

- we always add pre instructions
- if the task is the last task, we add post instructions as well.

Replacements and placeholders:

It is possible to have dynamic values inside of the instructions html (pre instructions only for now) to make instruction strings more generic. Following placeholders will be replaced with the according value on runtime:

- `{{duration}}` => exposureDuration(context task) or timeout(presenter task) of the task.
- `{{name}}` => adName of the task (context tasks only).
- `{{context}}` => context of the task (context tasks only).

## Examples:

### Simple example:

The following would show a custom instruction before a task but none after. It would also replace `{{duration}}` with the set exposure duration.

```json
"instructions": {
  "pre": {
    "html": "You will now see a Facebook newsfeed. Please browse it for the next {{duration}}s as you would normally do.",
    "button": "continue"
  }
}
```

[View in explorer](https://portal.incontext-research.com/docs/explorer?query=mutation+%28%24data%3A+ProjectInput%21%29+%7B%0A++projectCreate%28data%3A+%24data%29+%7B%0A++++id%0A++++createdBy+%7B%0A++++++name%0A++++%7D%0A++++subjectGroups+%7B%0A++++++nodes+%7B%0A++++++++id%0A++++++++entryUrl%0A++++++%7D%0A++++%7D%0A++%7D%0A%7D%0A&variables=%7B%0A++%22data%22%3A+%7B%0A++++%22description%22%3A+%22only+pre+instructions%22%2C%0A++++%22language%22%3A+%22en_US%22%2C%0A++++%22billingReference%22%3A+%22b+nr%22%2C%0A++++%22subjectGroups%22%3A+%5B%0A++++++%7B%0A++++++++%22device%22%3A+%22desktop%22%2C%0A++++++++%22contextMode%22%3A+%22public%22%2C%0A++++++++%22tasks%22%3A+%5B%0A++++++++++%7B%0A++++++++++++%22type%22%3A+%22context%22%2C%0A++++++++++++%22contextType%22%3A+%22facebookNewsfeedVideoAutoplay%22%2C%0A++++++++++++%22instructions%22%3A+%7B%0A++++++++++++++%22pre%22%3A+%7B%0A++++++++++++++++%22html%22%3A+%22You+will+now+see+a+Facebook+newsfeed.%3Cbr%2F%3E+Please+browse+it+for+the+next+%7B%7Bduration%7D%7Ds+as+you+would+normally+do.%22%2C%0A++++++++++++++++%22button%22%3A+%22continue%22%0A++++++++++++++%7D%0A++++++++++++%7D%2C%0A++++++++++++%22adName%22%3A+%22name+of+the+ad%22%2C%0A++++++++++++%22userGuidance%22%3A+false%2C%0A++++++++++++%22exposureDuration%22%3A+45%2C%0A++++++++++++%22timeout%22%3A+90%2C%0A++++++++++++%22data%22%3A+%7B%0A++++++++++++++%22text%22%3A+%22Join+us+today+at+redcross.org%22%2C%0A++++++++++++++%22brandName%22%3A+%22American+Red+Cross%22%2C%0A++++++++++++++%22mediaSrc%22%3A+%7B%0A++++++++++++++++%22videoUrl%22%3A+%22https%3A%2F%2Fd2dom67l5t3ck7.cloudfront.net%2Fdemos%2FAmericanRedCross15.mp4%22%0A++++++++++++++%7D%2C%0A++++++++++++++%22logoSrc%22%3A+%7B%0A++++++++++++++++%22imageUrl%22%3A+%22https%3A%2F%2Fd2dom67l5t3ck7.cloudfront.net%2Fdemos%2FredcrosslogoUS.jpg%22%0A++++++++++++++%7D%2C%0A++++++++++++++%22brandUrl%22%3A+%22https%3A%2F%2Fwww.facebook.com%2Fredcross%2F%22%0A++++++++++++%7D%0A++++++++++%7D%0A++++++++%5D%0A++++++%7D%0A++++%5D%0A++%7D%0A%7D)

![custom-instructions](https://portal.incontext-research.com/images/docs/custom-instructions.png)

### Complex example

```jsonc
// first task:
"instructions": {},
// second task:
"instructions": {
  "pre": {
    "html": "You will now see a Facebook newsfeed.<br/> Please browse it for the next {{duration}}s as you would normally do.",
    "button": "continue"
  },
  "post": {
    "html": "That was nice, wasn't it?",
    "button": "continue"
  }
}
// third task:
"instructions": null // null|undefined
```

This example shows no instructions in the first, custom instructions for the second, and default instructions in the third task.

[View in explorer](https://portal.incontext-research.com/docs/explorer?query=mutation+%28%24data%3A+ProjectInput%21%29+%7B%0A++projectCreate%28data%3A+%24data%29+%7B%0A++++id%0A++++createdBy+%7B%0A++++++name%0A++++%7D%0A++++subjectGroups+%7B%0A++++++nodes+%7B%0A++++++++id%0A++++++++entryUrl%0A++++++%7D%0A++++%7D%0A++%7D%0A%7D%0A&variables=%7B%0A++%22data%22%3A+%7B%0A++++%22description%22%3A+%22complext+instructions%22%2C%0A++++%22language%22%3A+%22en_US%22%2C%0A++++%22billingReference%22%3A+%22b+nr%22%2C%0A++++%22subjectGroups%22%3A+%5B%0A++++++%7B%0A++++++++%22device%22%3A+%22desktop%22%2C%0A++++++++%22contextMode%22%3A+%22public%22%2C%0A++++++++%22tasks%22%3A+%5B%0A++++++++++%7B%0A++++++++++++%22type%22%3A+%22context%22%2C%0A++++++++++++%22contextType%22%3A+%22facebookNewsfeedVideoAutoplay%22%2C%0A++++++++++++%22instructions%22%3A+%7B%7D%2C%0A++++++++++++%22adName%22%3A+%22name+of+the+ad%22%2C%0A++++++++++++%22userGuidance%22%3A+false%2C%0A++++++++++++%22exposureDuration%22%3A+45%2C%0A++++++++++++%22timeout%22%3A+90%2C%0A++++++++++++%22data%22%3A+%7B%0A++++++++++++++%22text%22%3A+%22Join+us+today+at+redcross.org%22%2C%0A++++++++++++++%22brandName%22%3A+%22American+Red+Cross%22%2C%0A++++++++++++++%22mediaSrc%22%3A+%7B%0A++++++++++++++++%22videoUrl%22%3A+%22https%3A%2F%2Fd2dom67l5t3ck7.cloudfront.net%2Fdemos%2FAmericanRedCross15.mp4%22%0A++++++++++++++%7D%2C%0A++++++++++++++%22logoSrc%22%3A+%7B%0A++++++++++++++++%22imageUrl%22%3A+%22https%3A%2F%2Fd2dom67l5t3ck7.cloudfront.net%2Fdemos%2FredcrosslogoUS.jpg%22%0A++++++++++++++%7D%2C%0A++++++++++++++%22brandUrl%22%3A+%22https%3A%2F%2Fwww.facebook.com%2Fredcross%2F%22%0A++++++++++++%7D%0A++++++++++%7D%2C%0A++++++++++%7B%0A++++++++++++%22type%22%3A+%22context%22%2C%0A++++++++++++%22contextType%22%3A+%22facebookNewsfeedVideoAutoplay%22%2C%0A++++++++++++%22instructions%22%3A+%7B%0A++++++++++++++%22pre%22%3A+%7B%0A++++++++++++++++%22html%22%3A+%22You+will+now+see+a+Facebook+newsfeed.%3Cbr%2F%3E+Please+browse+it+for+the+next+%7B%7Bduration%7D%7Ds+as+you+would+normally+do.%22%2C%0A++++++++++++++++%22button%22%3A+%22continue%22%0A++++++++++++++%7D%2C%0A++++++++++++++%22post%22%3A+%7B%0A++++++++++++++++%22html%22%3A+%22That+was+nice%2C+wasn%27t+it%3F%22%2C%0A++++++++++++++++%22button%22%3A+%22continue%22%0A++++++++++++++%7D%0A++++++++++++%7D%2C%0A++++++++++++%22adName%22%3A+%22name+of+the+ad%22%2C%0A++++++++++++%22userGuidance%22%3A+false%2C%0A++++++++++++%22exposureDuration%22%3A+45%2C%0A++++++++++++%22timeout%22%3A+90%2C%0A++++++++++++%22data%22%3A+%7B%0A++++++++++++++%22text%22%3A+%22Join+us+today+at+redcross.org%22%2C%0A++++++++++++++%22brandName%22%3A+%22American+Red+Cross%22%2C%0A++++++++++++++%22mediaSrc%22%3A+%7B%0A++++++++++++++++%22videoUrl%22%3A+%22https%3A%2F%2Fd2dom67l5t3ck7.cloudfront.net%2Fdemos%2FAmericanRedCross15.mp4%22%0A++++++++++++++%7D%2C%0A++++++++++++++%22logoSrc%22%3A+%7B%0A++++++++++++++++%22imageUrl%22%3A+%22https%3A%2F%2Fd2dom67l5t3ck7.cloudfront.net%2Fdemos%2FredcrosslogoUS.jpg%22%0A++++++++++++++%7D%2C%0A++++++++++++++%22brandUrl%22%3A+%22https%3A%2F%2Fwww.facebook.com%2Fredcross%2F%22%0A++++++++++++%7D%0A++++++++++%7D%2C%0A++++++++++%7B%0A++++++++++++%22type%22%3A+%22context%22%2C%0A++++++++++++%22contextType%22%3A+%22facebookNewsfeedVideoAutoplay%22%2C%0A++++++++++++%22adName%22%3A+%22name+of+the+ad%22%2C%0A++++++++++++%22userGuidance%22%3A+false%2C%0A++++++++++++%22exposureDuration%22%3A+45%2C%0A++++++++++++%22timeout%22%3A+90%2C%0A++++++++++++%22data%22%3A+%7B%0A++++++++++++++%22text%22%3A+%22Join+us+today+at+redcross.org%22%2C%0A++++++++++++++%22brandName%22%3A+%22American+Red+Cross%22%2C%0A++++++++++++++%22mediaSrc%22%3A+%7B%0A++++++++++++++++%22videoUrl%22%3A+%22https%3A%2F%2Fd2dom67l5t3ck7.cloudfront.net%2Fdemos%2FAmericanRedCross15.mp4%22%0A++++++++++++++%7D%2C%0A++++++++++++++%22logoSrc%22%3A+%7B%0A++++++++++++++++%22imageUrl%22%3A+%22https%3A%2F%2Fd2dom67l5t3ck7.cloudfront.net%2Fdemos%2FredcrosslogoUS.jpg%22%0A++++++++++++++%7D%2C%0A++++++++++++++%22brandUrl%22%3A+%22https%3A%2F%2Fwww.facebook.com%2Fredcross%2F%22%0A++++++++++++%7D%0A++++++++++%7D%0A++++++++%5D%0A++++++%7D%0A++++%5D%0A++%7D%0A%7D)

# List of supported languages

The current list of supported languages per context can be found here: [http://e2.ms/local](http://e2.ms/local)

# List of supported operating systems and browsers

The current list of supported OSs and browsers can be found here: [http://e2.ms/surveyintegration](https://docs.google.com/spreadsheets/d/1yBUl_M2__zjIBLzKlIzTb-klbmVsocPpdqBs_TJV3-4/edit#gid=1125839800)

## Overwriting defaults:

The supported Operating Systems and browsers can be overwritten using `deploymentFilters`, either on the subjectGroup or project level, in that order of precedence.

### Examples:

Allow Windows (Xp and above), Linux (any version) and any browser.

```json
"deploymentFilters": {
  "os": [
    {
      "name": "windows",
      "min": "5.2"
    },
    {
      "name": "linux"
    }
  ]
}
```

The above is equivalent to this:

```json
"deploymentFilters": {
  "os": [
    {
      "name": "windows",
      "min": "5.2"
    },
    {
      "name": "linux"
    }
  ],
  "browsers": [ ]
}
```

Example with browsers:

```json
"deploymentFilters": {
  "os": [
    {
      "name": "mac os",
      "min": "10.6"
    },
  ]
  "browsers": [
    {
      "name": "chrome",
      "min": "41"
    },
    {
      "name": "firefox",
      "min": "51"
    }
  ]
}
```

Matches are case insensitive, see [ua-parser-js](https://github.com/faisalman/ua-parser-js) for possible OS and browser name values.

# Query test results

Metrics and timelines of a project can be queried through the api. Both are part of a project response, so you can query metrics and timelines on every project object.

## Query filters

Both, timelines and metrics let you set filter for retrieving subsets of the recorded data.

- subjectGroup: String, if set only returns data for that subjectGroup inside a project
- task: String, if set, only returns data for a matching taskId. Note: if you don't set a subjectGroup but a taskId you may recieve entries for different subjectGroups.
- subject: Only returns data of a specific subject.

All filter can be combined together. [View in Explorer](https://portal.incontext-research.com/docs/explorer?query=%7B%0A++projects%28first%3A+1%29+%7B%0A++++nodes+%7B%0A++++++id%0A++++++metrics%28first%3A+10%2C+subjectGroup%3A+%22subjectGroupId%22%2C+task%3A+%22taskId%22%2C+subject%3A+%22subjectId%22%29+%7B%0A++++++++nodes+%7B%0A++++++++++projectId%0A++++++++++subjectGroupId%0A++++++++++taskId%0A++++++++++subjectId%0A++++++++%7D%0A++++++%7D%0A++++++timelines%28first%3A+10%2C+subjectGroup%3A+%22subjectGroupId%22%2C+task%3A+%22taskId%22%2C+subject%3A+%22subjectId%22%29+%7B%0A++++++++nodes+%7B%0A++++++++++projectId%0A++++++++++subjectGroupId%0A++++++++++taskId%0A++++++++++subjectId%0A++++++++%7D%0A++++++%7D%0A++++%7D%0A++%7D%0A%7D%0A&variables=)

## Metrics

A metric is a (ad/media) preformance indicator derived of events of a recording. It typically measures things like durations a video was seen, the time a button was clicked.

Every Metric Set contains one or more elementSets in which metrics are grouped. An element describes an observed element during a test run. In most cases it's the tested ad and its properties (like video controls).

## Timelines

A timeline is a sequence of recorded events.

# Contact

For technical support regarding this API please send an email to
api-support@eye-square.com or ask us in slack.
