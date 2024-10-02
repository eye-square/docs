<h1 align="center">
  <img src="../images/logo.png" alt="in-context" width="75" height="75">
</h1>

# in-context plugins

In-context plugins are a streamlined way to extend in context functionality. 

Following features are available for a plugin to use:

- [client scripts](./client-scripts-api.md): Run scripts with powerful hooks, which can extend in-context browsing behavior. This can be anything from simple debugging tools to advanced emotion and/or eye tracking. 
- [webhooks](./webooks.md): Webhooks allow your servers to react to events that happen within in-context. This makes you able to get notified whenever a plugin has been configured in a project to react on updates of individual browsing sessions.
- options: A plugin can define options that can be used by the server(for example api secrets) and client (for example script options). Each option can be set on organization level and all their projects and individually on project/subject group level.
- client capabilities: A plugin can define which browsers and oss it supports. In context will take this into account before letting subjects enter the system.
