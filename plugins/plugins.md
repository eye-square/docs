<h1 align="center">
  <img src="../images/logo.png" alt="in-context" width="75" height="75">
</h1>

# in-context plugins

In-context plugins are a streamlined way to extend in context functionality. 

Following features are available for a plugin to use:

- [client scripts](./client-scripts-api.md): Run scripts with powerful hooks to hook into and change in context browsing behavior. This can be anything from simple debugging tools to advanced emotion and/or eye tracking. 
- [webhooks](./webooks.md): Webhooks allow you do react on your server side to events that happen within in-context. From getting notified whenever a plugin has been configured in a project too updates on individual browsing sessions.
- options: A plugin can define options to use server(for example api secrets) and client side (for example script options). Each option can be set for a whole organization and all their projects of on a project/subject group basis.
- client capabilities: A plugin can define which browsers and oss it supports. In context will take this into account before letting subjects enter the system.
