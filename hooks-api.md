# in-context hook api

hooks are a means to attach jobs at specific moments of a task's lifecycle. 

The jobs run on the same context as the displayed content, typically that would be the top frame. They are also blocking: if a promise is returned, the internal job runner will wait for it to resolve before advancing to he next task stage

Two types of hook can be set using:
* `setInitHook(fn)`: job will run only once, the first time the script is used, before the task begin.
* `setTaskHook(stage, fn)`: job runs on each task.

This is a sample application that loads an external sdk and calls its methods. For brevity, we assume two functions `createScript` and `waitFor` that load a script and wait until a condition is fulfilled:

```javascript
window.setInitHook(() => { 
  // load our bundle
  await createScript('https://extern.integration.com');
  // our script creates an SDK object globally, wait for it
  await waitFor(() => window.SDK);
});

window.setTaskHook('preparation', () => window.SDK.prepareUser());

// use eye-square session id to identify this session
window.setTaskHook('recording', ({ context }) => window.SDK.startRecording(context.session.id));

window.setTaskHook('completion', () => window.SDK.stopRecording());
```

## jobs

A job is just a function that receives a single argument: 
```javascript
window.setTaskHook('recording', ({ context, channel, state }) => { });
```

The argument consists of the following fields

- `context` is an immutable object that holds the current task and session configuration.

- `channel` is used to send and receive messages using an interface similar to a [broadcast channel](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API):
  ```javascript
  window.setTaskHook("recording", ({ channel }) => {
    channel.onmessage = (event) => {
      if (event.type === 'video-pause') {
        // react to the target video being paused
      }
    };
  });
  ```

- `state` is used to share state between jobs. All jobs can return an object that will be merged with the current state and available for all subsequent jobs. Scripts are free to use the global namespace, but we recommend using the `state` object to avoid conflicts and keep things tidy.
  ```javascript
  window.setInitHook(({ state }) => ({ taskCount: 0 })); 

  window.setTaskHook("recording", ({ state }) => ({
    taskCount: state.taskCount + 1 
  }));

  window.setTaskHook("completion", ({ state }) => {
    console.log('tasks completed:', state.taskCount)
  });
  ```
  In a two task setup, the script above should print:
  ```log
  tasks completed: 1
  tasks completed: 2
  ```

- `render` is a utility function for rendering custom content. It's meant as a way to help hide and manage the rendered content after each stage
  ```javascript
  window.setTaskHook('preparation', ({ render }) =>
    new Promise((resolve) => {
      const warningDiv = document.createElement('div');
      warningDiv.innerText = 'hey, watch out!';

      // the div will be automatically shown and hidden after 2 seconds
      render(div);
      setTimeout(resolve, 2000);
    })
  );
  ```
