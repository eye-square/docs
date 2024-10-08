<h1 align="center">
  <img src="../images/logo.png" alt="in-context" width="75" height="75">
</h1>

# in-context scripts api

The in-context scripts api allows you to run custom scripts inside an in-context app. The use cases are many, but some examples include:

- Loading external SDKs
- Adding extra instructions or preparation steps
- Sending extra variables to the survey
- Listening to events and storing them for later analysis
- Modifying the DOM during the task, for example to highlight elements or add overlays
- Running custom logic at the end of a task

For this, we provide we provide a very simple API that allows you to attach jobs at specific moments of a task's lifecycle. The jobs are essentially just async functions that receive all session information and common utilities useful for the cases above.

# setting up hooks

The jobs run on the same context as the displayed content, typically that would be the top window frame. Hooks are **blocking**: if a promise is returned, it will be waited for before advancing to he next task stage. An exception to this is the `recording` stage, the duration of the actual task is defined in the project and can't be modified by the script.

Three types of hook can be set using:

- `setInitHook(fn)`: job will run only once, the first time the script is used, before the task or tasks begin.

- `setTaskHook(stage, fn)`: job runs on each task.

- `setTeardownHook(fn)`: job will run only once, before we navigate away from the in-context app to go back to a survey.

This is a sample application that loads an external sdk and calls its methods:

```javascript
window.setInitHook(async ({ utils }) => {
	// load an external sdk
	await utils.createScript('https://external.integration.com');
	// maybe wait for the sdk to be available
	await utils.waitFor(() => window.SDK);
});

window.setTaskHook('preparation', () => window.SDK.prepareUser());

window.setTaskHook('recording', ({ context }) =>
	// you can use the session id to map your data to the current session
	window.SDK.startRecording(context.session.id)
);

window.setTaskHook('completion', () => window.SDK.stopRecording());

window.setTeardownHook(async ({ utils }) => {
	const sdk = await utils.waitFor(() => window.SDK);
	await sdk.cleanup();
});
```

In the example above, applied to a two task setup, the task hooks would run twice, once for each task. The init hook would run only once, before the first task, and the teardown hook would run only once, after the second task.

## jobs

A job is just a function that receives a single argument:

```javascript
window.setTaskHook('recording', ({ context, channel, state }) => {});
```

The argument consists of the following fields

- `context` is an immutable object that holds the current task and session configuration.

- `channel` is used to send and receive messages using an interface similar to a [broadcast channel](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API):

  ```javascript
  window.setTaskHook('recording', ({ channel }) => {
  	// Add event listener
  	channel.on((event) => {
  		// Handle events coming in through the channel
  	});
  });
  ```

  For more info on the events received and ways to filter for them, see the [event objects section](#event-objects).

- `state` is used to share state between jobs. All jobs can return an object that will be merged with the current state and available for all subsequent jobs. Scripts are free to use the global namespace, but we recommend using the `state` object to avoid conflicts and keep things tidy.

  ```javascript
  window.setInitHook(({ state }) => ({ taskCount: 0 }));

  window.setTaskHook('recording', ({ state }) => ({
  	taskCount: state.taskCount + 1,
  }));

  window.setTaskHook('completion', ({ state }) => {
  	console.log('tasks completed:', state.taskCount);
  });
  ```

  In a two task setup, the script above should print:

  ```log
  tasks completed: 1
  tasks completed: 2
  ```

- `render` is a utility function for rendering custom content. It's meant as a way to help hide and manage the rendered content after each stage

  ```javascript
  window.setTaskHook(
  	'preparation',
  	({ render }) =>
  		new Promise((resolve) => {
  			const warningDiv = document.createElement('div');
  			warningDiv.innerText = 'hey, watch out!';

  			// the div will be automatically shown and hidden after 2 seconds
  			render(div);
  			setTimeout(resolve, 2000);
  		})
  );
  ```

- `utils` commonly used utility functions.

  ```javascript
  /*
    - sleep
    - waitFor, waitForElement, waitForElements
    - createScript, createStyles, injectStyles
    - showMessage
  */
  window.setTaskHook('preparation', async ({ utils }) => {
    // wait half a second
    await utils.sleep(500);

    // create a script tag from a source url
    await utils.createScript('https://external.com/sdk.js');

    // create a style tag from a source url
    await utils.createStyles('https://external.com/styles.css');

    // create a style tag from a style string
    utils.injectStyles(`
      #test-id {
        border: 1px solid red;
      }
    `);

    // wait for a condition to evaluate truthy. optional arguments: interval(ms), timeout(ms)
    await utils.waitFor(() => window.SDK /*, 500, 5000 */);

    // wait for an element using css selectors
    const element = await utils.waitForElement('#test-id');

    // wait for multiple elements using css selectors
    const elements = await utils.waitForElements('a.selected');

    // a key / value pair that will be added to the session params and reported back to the survey
    await utils.setSessionParam('consentFormAccepted', '1');

    // Display a centered message and wait for the button to be clicked
    await utils.showMessage({ message: `Ok, let's start.<br><br>Press continue!` button: 'Continue' });

    // Display a ok/cancel dialog
    // todo change this to promise
    const accepted = await utils.confirm({ message: 'We would like to access your camera' confirmText: 'Ok', rejectText: 'Cancel'  });
  });
  ```

# event objects

An event object will have the following structure:

```javascript
{
  // type of the event
  type: 'mediaPause',
  // UTC timestamp of the moment when the event occurred
  timestamp: 1666266197290,
  // event source information
  origin: {
    path: ['newsfeed', 'post', 0, 'video'],
    trackingId: 'value', // optional
    domElement: videoDomElement, // optional (in case there is a DOM element related to the event)
    tracked: true
  },
  // event data/body
  payload: { }
}
```

The `origin.path` property of an event refers to the abstract hierarchy of the context, not a location within the dom tree. The fields in `payload` are variable and might change from event to event.

The `origin.domElement` property should be used only when you can't get the needed information from the event itself.
In case you you think there is information that should be added to the event, please contact us.

## Filtering events

As the channel is handling a lot of different events you most of the time need to filter the events to
react to only the events that you need to. There are multiple properties that are of interest for you.

For now you can filter by using an `if` clause:

```javascript
window.setTaskHook('recording', ({ channel }) => {
	channel.on((event) => {
		if (event.origin.tracked && event.type === 'mediaPause') {
			// react to the target video being paused
		}
	});
});
```

### `origin.tracked`

The most important property to filter by is most probably `origin.tracked`, which defines if the the
event is happening on an element of interest (e.g. a video that a study wants to test with).

### `type`

The event type is the next important information that you will often need to filter on. Below you can find
a section with all the events we currently support.

### `origin.path`

The origin path describes the position of the element the event happened on in the content hierarchy.
Often you want to only want to look at the last segment of that path as it describes the element the
event has happened on (e.g. a `video` or `static` (for images) in a post).

An example of a video in a post:

```jsonc
{
	"type": "mediaStart",
	"origin": {
		"path": ["feed", "post", 1, "video"]
	}
}
```

The above describes that the event happened on a video element in the second post of the news- or videofeed.

### `origin.trackingId`

The tracking ID is a unique identifier for an entry that is tracked. This can be a post in Facebook
or a story on instagram that a study wants to test on. It's the same for all events of sub elements
of this entry (a video of a tracked post has the same id as the like button for example).

## Event list

This is a list of all currently supported events (more are coming later):

### `taskStart` / `taskEnd`

Global events triggered at the beginning and right before the end of each task.

Payload:

```jsonc
{}
```

### `mediaStart`

Play/start of a video or audio.

Payload:

```jsonc
{
	"muted": true, // mute state of the media at start time
	"currentTime": 12.34 // current time of the media in seconds the moment the event happened
}
```

### `mediaPause`

Mid playtime pause of a video or audio (excluding the video ending).

Payload: same as `mediaStart` (same for all media events)

```jsonc
{
	"muted": true, // mute state of the media at start time
	"currentTime": 12.34 // current time of the media in seconds the moment the event happened
}
```

### `mediaEnded`

Video or audio has ended (`currentTime` === `duration`).

```jsonc
{
	"muted": true, // mute state of the media at start time
	"currentTime": 12.34 // current time of the media in seconds the moment the event happened
}
```

### `mediaTimeUpdate`

Fires when the media's `currentTime` attribute is updated.

```jsonc
{
	"muted": true, // mute state of the media at start time
	"currentTime": 12.34 // current time of the media in seconds the moment the event happened
}
```

### `visibilityChange`

An element changes visibility ratio between 0 (not visible at all) and 1 (fully visible).

Payload:

```jsonc
{
	"visibilityRatio": 0.545, // Ratio of how much the element is visible/in the viewport
	"visibleRect": {
		"x": 100, // X position in the viewport
		"y": 200, // Y position in the viewport
		"width": 340, // Width of the visible part of the element in pixel
		"height": 340 // Height of the visible part of the element in pixel
	}
}
```

If the element is not visible, the value of `visibleRect` is `null`.

### `toggle`

This event is used to express changes in boolean values.

Payload:

```jsonc
{
	"value": true // true | false;
}
```

### `routeChange`

This event is triggered when a change in the internal router happens.

Payload:

```jsonc
{
	"location": {
		"pathname": "/products",
		"search": "?page=0",
		"query": {
			"page": 0
		},
		"hash": "#"
	},
	"action": "PUSH" // PUSH, REPLACE
}
```

### `searchTermChange`

Subject changed and applied a new search term. Includes also the result count for the new search.

Payload:

```jsonc
{
	"searchTerm": "chocolate",
	"resultCount": 145
}
```

The property `resultCount` is optional and could be omitted if result count is not known or not available.

### `filterChange`

Subject changed the filters for results and applied the filters or initial empty filter has been applied.

Payload:

```jsonc
{
	"activeFilters": {
		"brandName": ["snickers", "milka", "lindt", "m&ms"]
	},
	"resultCount": 46 // Results after filter has been applied
}
```

### `paginationChange`

Results page has been changed or initialized. Origin of event tells where the pagination has changed.

Payload:

```jsonc
{
	"hasNextPage": true,
	"hasPrevPage": true,
	"page": 6,
	"totalPageCount": 59
}
```

### e-commerce specific events

#### `ecomCheckout`

Subject checks out and ends their shopping journey. Includes the products that were in the basket in
the time of the checkout.

Payload:

```jsonc
{
	"basketContents": [
		{
			"id": "1234", // Product ID
			"price": 1.99,
			"quantity": 6,
			"sources": ["results-page", "details-page", "details-page-related"], // Where the products were added from
			"time": 12345 // Time in ms after session start of the first addition
		}
	],
	"total": 11.94,
	"totalDiscounted": 11.43,
	"discountApplied": true
}
```

#### `ecomBasketQuantityChange`

Subject changes the quantity of a product, either by adding it to the basket for the first time,
removing it from the basket or changing the quantity explicitly.

Payload:

```jsonc
{
	"id": "1234", // Product ID
	"quantity": 1,
	"quantityDelta": 1,
	"productTotal": 4.99,
	"basketTotal": 23.52,
	"source": "checkout" // Where the quantity was changed at
}
```
