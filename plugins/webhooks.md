<h1 align="center">
  <img src="../images/logo.png" alt="in-context" width="75" height="75">
</h1>

# webhooks

In-context provides plugins the options to listen to webhooks in order to react on events that happen within in-context.

To setup a webhook you need to provide an `http POST` endpoint that in-context then calls when system events occur. The call includes information about the event that can be used to process it.

*example webhook definition*

```
 webhook: {
    url: 'https://my.webhook.url/endpoint', // POST endpoint in-context will send system events too. 
    permissions: ['projectUpdated', 'projectCreated', 'sessionCreate', 'sessionFinish'], // the events/permissions the plugin wants to get notified about
    secret: 'abc', // a secret in-context uses to sign the webhook payload with, so you can make sure it's in-context that calls the endpoint
  },
  ```

## Validating webhook deliveries

In-context will use your secret to create a hash signature that's sent to you with each payload. The hash signature will appear in each delivery as the value of the e2-signature-256 http header.

In your code that handles webhook deliveries, you should calculate a hash using your secret. Then, compare the hash that In-context sent with the expected hash that you calculated, and ensure that they match.

There are a few important things to keep in mind when validating webhook payloads:

In-context uses an HMAC hex digest to compute the hash.
The hash signature always starts with sha256=.
The hash signature is generated using your webhook's secret and the payload contents.
If your language and server implementation specifies a character encoding, ensure that you handle the payload as UTF-8. Webhook payloads can contain unicode characters.

### Testing the webhook payload validation

You can use the following secret and payload values to verify that your implementation is correct:

secret: "It's a Secret to Everybody"
payload: "Hello, World!"
If your implementation is correct, the signatures that you generate should match the following signature values:

signature: 757107ea0eb2509fc211221cce984b8a37570b6d7586c22c46f4379c8b043e17
e2-signature-256: sha256=757107ea0eb2509fc211221cce984b8a37570b6d7586c22c46f4379c8b043e17

## Possible events and their payloads

The following webhook events are possible: 

- projectCreated: A project has been created with at least one subject group that is using your plugin.
- projectUpdated: A project has been updated with at least one subject group that is using your plugin after the update.
- sessionCreate: A subject has started a test/session with your plugin being used.
- sessionFinish: A subject has finished a test/session with your plugin being used.

### projectCreated and projectUpdated

Both projectCreated and projectUpdated share the same payload, it includes the project with all their subject groups that your plugin is used in. It also sends information about the user (organization) the project belongs to.

An example payload looks like this:

```
{
    event: {
      type: 'projectCreated|projectUpdated',
    },
    organization: {
      name: 'eye-square',
      code: 'ES',
    },
    project: {
      id: '2021-01_ES001',
      language: 'en-US',
      deploymentFilters: null,
      subjectGroups: [
        {
          id: 'aSubjectGroup',
          device: 'desktop',
          deploymentFilters: null,
          tasks: [
            {
              id: '1',
              type: 'context',
              contextType: 'facebook-newsfeed-video-autoplay',
              userGuidance: false,
              instructions: {
                pre: {
                  html: '\n  <h2 style="margin-bottom: 25px;">\n    \rYou will now be taken to Facebook. <br> Please look at the page as you would naturally. <br><br> After 90 seconds the survey will automatically continue.\r\n\n  </h2>\n',
                  button: 'Continue',
                },
                post: null,
              },
              timeout: 90000,
              adName: 'google windy',
              data: {
                text: 'Placeholder for your post text',
                brandName: 'eye square brand',
                brandUrl: 'http://test-page:1337/brand.html',
                mediaSrc: {
                  videoUrl: 'http://test-page:1337/SampleVideo_1280x720_1mb.mp4',
                  length: 5.312,
                },
                trackingId: 'aSubjectGroup',
                trackingType: ['facebook-post-video'],
              },
              successParameters: {
                visible: 1,
              },
            },
            {
              id: '2',
              type: 'context',
              contextType: 'facebook-newsfeed-video-autoplay',
              userGuidance: true,
              instructions: {
                pre: {
                  html: '\n  <h2 style="margin-bottom: 25px;">\n    \rWe would like you to view Facebook again.\r\nThis time please watch the <span style="font-weight:bold;">google windy ad on Facebook </span>.<br><br>\r\nAfter 45 seconds the survey will continue.\n  </h2>\n',
                  button: 'Continue',
                },
                post: {
                  html: '\n  <h2 style="margin-bottom: 25px;">\n    \rPlease click the button below to continue.\r\n \n  </h2>\n',
                  button: 'Continue',
                },
              },
              timeout: 90000,
              exposureDuration: 45000,
              adName: 'google windy',
              data: {
                text: 'Placeholder for your post text',
                brandName: 'eye square brand',
                brandUrl: 'http://test-page:1337/brand.html',
                mediaSrc: {
                  videoUrl: 'http://test-page:1337/SampleVideo_1280x720_1mb.mp4',
                  length: 5.312,
                },
                trackingId: 'aSubjectGroup',
                trackingType: ['facebook-post-video'],
              },
              successParameters: {
                elements: [
                  {
                    id: 'testId',
                    loaded: true,
                  },
                ],
                loaded: true,
              },
            },
          ],
          plugins: [
            {
              name: 'test',
              options: {
                requiredOrgField: true,
                requiredProjectField: true,
              },
            },
          ],
        },
      ],
    },
  }
  ```

  ### sessionStart and sessionFinish

  Both events share the same payload:

  ```
{
    event: {
      type: 'sessionStart|sessionFinish',
    },
    session: {
      id: 'in-context-session-id',
      subjectId: 'subject-id-alias-user-hash',
      result: 'none',
      createdAt: 'date string',
      updatedAt: 'date string',
    },
    organization: {
      name: 'eye-square',
      code: 'ES',
    },
    project: { // top level project properties, no groups included!
      id: '2021-01_ES001',
      language: 'en-US',
      deploymentFilters: null,
    },
    subjectGroup: { // subject group of the session/subject
      id: 'aSubjectGroup',
      device: 'desktop',
      deploymentFilters: null,
      tasks: [
        {
          id: '1',
          type: 'context',
          contextType: 'facebook-newsfeed-video-autoplay',
          userGuidance: false,
          instructions: {
            pre: {
              html: '\n  <h2 style="margin-bottom: 25px;">\n    \rYou will now be taken to Facebook. <br> Please look at the page as you would naturally. <br><br> After 90 seconds the survey will automatically continue.\r\n\n  </h2>\n',
              button: 'Continue',
            },
            post: null,
          },
          timeout: 90000,
          adName: 'google windy',
          data: {
            text: 'Placeholder for your post text',
            brandName: 'eye square brand',
            brandUrl: 'http://test-page:1337/brand.html',
            mediaSrc: {
              videoUrl: 'http://test-page:1337/SampleVideo_1280x720_1mb.mp4',
              length: 5.312,
            },
            trackingId: 'aSubjectGroup',
            trackingType: ['facebook-post-video'],
          },
          successParameters: {
            visible: 1,
          },
        },
        {
          id: '2',
          type: 'context',
          contextType: 'facebook-newsfeed-video-autoplay',
          userGuidance: true,
          instructions: {
            pre: {
              html: '\n  <h2 style="margin-bottom: 25px;">\n    \rWe would like you to view Facebook again.\r\nThis time please watch the <span style="font-weight:bold;">google windy ad on Facebook </span>.<br><br>\r\nAfter 45 seconds the survey will continue.\n  </h2>\n',
              button: 'Continue',
            },
            post: {
              html: '\n  <h2 style="margin-bottom: 25px;">\n    \rPlease click the button below to continue.\r\n \n  </h2>\n',
              button: 'Continue',
            },
          },
          timeout: 90000,
          exposureDuration: 45000,
          adName: 'google windy',
          data: {
            text: 'Placeholder for your post text',
            brandName: 'eye square brand',
            brandUrl: 'http://test-page:1337/brand.html',
            mediaSrc: {
              videoUrl: 'http://test-page:1337/SampleVideo_1280x720_1mb.mp4',
              length: 5.312,
            },
            trackingId: 'aSubjectGroup',
            trackingType: ['facebook-post-video'],
          },
          successParameters: {
            elements: [
              {
                id: 'testId',
                loaded: true,
              },
            ],
            loaded: true,
          },
        },
      ],
      plugins: [
        {
          name: 'test',
          options: {
            requiredOrgField: true,
            requiredProjectField: true,
          },
        },
      ],
    }
  }
  ```