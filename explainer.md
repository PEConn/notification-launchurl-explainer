# Notifications launchURL Explainer

## TL;DR

Add a `launchURL` option to notification creation that will allow browsers to respond to notification clicks without needing to execute JavaScript.

## Background

### How Notifications Work
There are two types of notifications, persistent notifications that have an associated service and non-persistent notifications that do not.
To display a persistent notification, [ServiceWorkerRegistration.showNotification][swr-show-notification] is used.
To display a non-persistent notification, a [Notification][notification] object is created and a notification is automatically displayed.

Both the `ServiceWorkerRegistration.showNotification` method `Notification()` take a title and a map of options, such as text for the notification body or an icon.
Notifications may also take a set of actions, specifying various buttons shown on the notification.

When a non-persistent notification is clicked, a callback set on the Notification object is called.
When a persistent notification is clicked, the associated service worker receives the `notificationclick` event.

In either case, the callback can perform arbitrary actions, for example launching a new URL or focusing an existing session.

|   | Persistent Notifications | Non-Persistent Notifications |
| --- | --- | --- |
| Display               | `ServiceWorkerRegistration.showNotification` | `Notification()` |
| Callback when clicked | `ServiceWorkerGlobalScope.onnotificationclick` | `Notification.onclick` |
| Callback when closed  | `ServiceWorkerGlobalScope.onnotificationclose` | `Notification.onclose` |

[swr-show-notification]: https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/showNotification
[notification]: https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification

### Why We Should Improve This

When a user clicks on a notification, the browser must execute some JavaScript before it can determine what to do.
This causes a delay and makes web notifications less responsive than their native counterparts, where reducing responsiveness is important for providing the user with [immediate feedback][response-limits] when clicking on a notification.

[response-limits]: https://www.nngroup.com/articles/response-times-3-important-limits/

### Launch Handler API

The proposed [launch_handler][launch-handler] manifest member allows web apps to customize how they are launched through declarative rules.
There are two different parts of this proposal.

* The `LaunchQueue` and `LaunchParams` interfaces allow existing browing contexts to receive information from new app launches.
* The `launch_handler` manifest member specifies how app launches are handled, answering these two questions:
  * If the user opens the webapp while an existing context exists, should a new browsing context be created or should the most recently active one be reused?
  * If an existing context is chosen, should a navigation be triggered or not (in which case the existing client can use the `LaunchQueue` and `LaunchParams` interfaces to be informed).

[launch-handler]: https://github.com/alancutter/manifest-incubations/blob/launch-handler/launch_handler-explainer.md

## Changes to the API

We propose adding the `launchURL` option to the `options` parameter in both `ServiceWorkerRegistration.showNotification` and `Notification()`.
(There is precedence for [this capitalization here][capitalization].)
When the notification is clicked, the browser can immediately launch a new tab to that URL before executing any JavaScript, and automatically close the notification.

The JavaScript callbacks will still be executed.
However, they will not be invoked with *user activation*, and will not be able to [focus existing][focus] or [open new][open] windows.

When the notification is interacted with, it will automatically be closed and the appropriate close callbacks will be invoked.

[capitalization]: https://github.com/WICG/content-index/issues/21
[focus]: https://w3c.github.io/ServiceWorker/#dom-windowclient-focus
[open]: https://w3c.github.io/ServiceWorker/#clients-openwindow

### Focusing existing windows

Often when a notification is triggered, the Service Worker will contain logic that checks to see if there are any open tabs, and if so will focus and navigate one of those tabs instead of launching a new one.
We can achieve this behaviour in a declarative manner by using the [Launch Handler API][launch-handler].

The options for `launch_handler` are:

* `route_to` - which browser context to open.
  * `new-client` - opens a new browser context.
  * `existing-client` - focuses the most recently used browser context.
  * `auto` - let the user agent decide.
* `navigate_existing_client` - what to do for the `existing-client` option.
  * `always` - navigate the browser context.
  * `never` - do not navigate, but the existing browser context can detect this through the other parts of the API.

We can add these options to the `options` parameter in `ServiceWorkerRegistration.showNotification` and `Notification()`, eg:

```javascript
var notification = new Notification('New message', {
  body: 'You have a new message',
  launchURL: 'https://www.example.com/inbox',
  launchHandler: {
    routeTo: 'existing',
    navigateExistingClient: 'never'
  }
})
```

### Notifications with no callbacks

With the `launchURL` option, simple notification use can achieved without any callbacks:

```javascript
var notification = new Notification('New message', {
  body: 'You have a new message',
  launchURL: 'https://www.example.com/inbox'
});
```

This notification will be displayed to the user, and when clicked, it will launch the given URL and close itself.

## Open Questions

### Support for Action Buttons

We could also add the launchURL field to the [NotificationAction][notification-action] interface, which would allow clicks on action buttons to also launch URLs immediately.

```javascript
var notification = new Notification('New message', {
  body: 'You have a new message',
  launchURL: 'https://www.example.com/inbox',
  actions: [{
    action: 'settings',
    title: 'Notification Settings',
    launchURL: 'https://www.example.com/user-settings#notifications'
  }]
});
```

[notification-action]: https://developer.mozilla.org/en-US/docs/Web/API/NotificationAction

### UX Guidelines

What guidelines can we provide to websites and push services to give the best user experience?

## Appendix

### Platform Specific Considerations

[Android 12][android] introduced changes that prevent a notification launching background tasks which lead to a user visible change.
Notifications can either trigger background computation or immediate user-visible changes.
Since the browser does not know whether a notification click will lead to a URL being launched or a window being focused without executing JavaScript, there is currently no way for web platform notifications to meet this standard.

[android]: https://developer.android.com/about/versions/12/behavior-changes-12#notification-trampolines
