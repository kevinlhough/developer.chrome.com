---
layout: 'layouts/doc-post.njk'
title: 'Handling events with a service worker'
seoTitle: 'Chrome extension service worker tutorial.'
subhead: 'This tutorial teaches concepts relevant to extension service workers.'
description: 'Learn how to create and debug an extension service worker.'
date: 2023-04-02
# updated: 2022-06-13
---

## Overview {: #overview }

This tutorial builds an extension that allows users to open Chrome API reference pages using the omnibox. It also provides a daily Chrome extension tip.

{% Video src="video/BhuKGJaIeLNPW9ehns59NfwqKxF2/WmVEGpEZ9ts1J0pUOzEr.mp4", width="600", height="398", autoplay="true", muted="true"%}

This article will cover the following topics:

- Registering a service worker and importing modules.
- Debugging your extension service worker.
- Managing state and handling events.
- Triggering periodic events.
- Communicating with content scripts.

## Before you start {: #prereq }

This guide assumes that you have basic web development experience. We recommend reviewing [Extensions 101][doc-ext-101] and [Development Basics][doc-dev-basics] for an introduction to extension development.

## Build the extension {: #build }

Start by creating a new directory called `quick-api-reference` to hold the extension files, or download the source code from our [GitHub samples][github-open-api] repo.

### Step 1: Register the service worker {: #step-1 }

Create the [manifest][doc-manifest] file in the root of the project and add the following code:

{% Label %}manifest.json:{% endLabel %}

```json/8-10
{
  "manifest_version": 3,
  "name": "Open extension API reference",
  "version": "1.0.0",
  "icons": {
    "16": "icon-16.png",
    "128": "icon-128.png"
  },
  "background": {
    "service_worker": "service-worker.js",
  },
}
```

Extensions register their service worker in the manifest, which only takes a single JavaScript file.
There's no need to use `navigator.serviceWorker.register()`, like you would in a web app. See
[Differences between extension and web service workers](tbd) to learn more.

You can download the icons located on the [Github repo][github-open-api].

### Step 2: Import multiple service worker modules {: #step-2 }

Our service worker implements two features. For better maintainability, we will implement each feature in a separate module. First, we need to declare the service worker as an [ES Module][mdn-es-module] in our manifest, which allows us to import modules in our service worker:

{% Label %}manifest.json:{% endLabel %}

```json/3-3
{
 "background": {
    "service_worker": "service-worker.js",
    "type": "module"
  },
}
```

Create the `service-worker.js` file and import two modules:

```js
import './sw-omnibox.js';
import './sw-tips.js';
```

Create these files and add a console log to each one.

{% Columns %}

{% Column %}

{% Label %}sw-omnibox.js:{% endLabel %}

```js
console.log("sw-omnibox.js")
```
{% endColumn %}

{% Column %}
{% Label %}sw-tips.js:{% endLabel %}

```js
console.log("sw-tips.js")
```
{% endColumn %}

{% endColumns %}

See [Importing scripts](tbd) to learn about other ways to import multiple files in a service worker.


{% Aside 'important' %}

Remember to set `type.module` when using a modern module bundler framework, such as [CRXjs Vite plugin][crxjs-vite].

{% endAside %}

### _Optional: Debugging the service worker_ {: #step-3 }

Let's quickly go over how to find the service worker logs and know when it has terminated. First, follow the instructions to [Load an unpacked extension][doc-dev-basics-unpacked]. After 30 seconds it will show "service worker(inactive)" which means the service worker has terminated. 

Click on the "service worker(inactive)" hyperlink to inspect it. 

{% Video src="video/BhuKGJaIeLNPW9ehns59NfwqKxF2/D1XRaA6q4xn9Ylwe1u1N.mp4", width="800", height="314", autoplay="true", muted="true", loop="true" %}

Did you notice that inspecting the service worker woke it up? That's right! Opening the service worker in the devtools will keep it active. This means, if you want to make sure that your extension behaves correctly when your service worker is terminated, remember to close the DevTools.

Now let's break the extension to learn where to locate errors. One way to do this is to delete the ".js" from the `'./sw-omnibox.js'` import in the `service-worker.js` file. Chrome will be unable to register the service worker.

Go back to chrome://extensions and refresh the extension. The following error will appear:

{% Video src="video/BhuKGJaIeLNPW9ehns59NfwqKxF2/AbMNDSbURLKjH1Jm1C9Q.mp4", width="400", height="477", autoplay="true", muted="true", loop="true" %}

See [Debugging extensions](tbd) for more ways debug the extension service worker.

{% Aside 'caution' %}
Don't forget to fix the file name before moving on!
{% endAside %}

### Step 4: Initialize the state {: #step-4 }

Extensions can save initial values to storage on installation. To use the [`chrome.storage`][api-storage] API, we need to request permission in the manifest:

{% Label %}manifest.json:{% endLabel %}

```json
{
  ...
  "permissions": ["storage"],
}
```

The following code save the default suggestions to storage when the extension is first installed by listening to the `onInstalled` event:

{% Label %}sw-omnibox.js:{% endLabel %}

```js
...
// Save default API suggestions
chrome.runtime.onInstalled.addListener(({ reason }) => {
  if (reason === 'install') {
    chrome.storage.local.set({
      apiSuggestions: ['tabs', 'storage', 'scripting']
    });
  }
});
```

Service workers do not have direct access to the [window object][mdn-window], therefore cannot use
[window.localStorage()][mdn-local-storage] to store values. Also, service workers are short-lived execution environments;
they get terminated repeatedly throughout a user's browser session, which makes it incompatible with
global variables.

See [Saving state](TBD) to learn about storage options for extension service workers.

### Step 5: Register your events {: #step-5 }

All event listeners need to be statically registered in the global scope of the service worker. In other words, event listeners should not be nested in async functions. This way Chrome can ensure that all event handlers are restored in case of a service worker reboot.

To use the [`chrome.omnibox`][api-omnibox] API first add the omnibox keyword to the manifest:

{% Label %}manifest.json:{% endLabel %}

```json
{
  ...
  "minimum_chrome_version": "102",
  "omnibox": {
    "keyword": "api"
  },
}
```

{% Aside 'important' %}
The [`"minimum_chrome_version"`][manifest-min-version] explains how this key behaves when a user tries to install your extension but isn't using a compatible version of Chrome.

{% endAside %}

The following code registers the omnibox event listeners at the top level of the script and updates [chrome.storage][api-storage] with the most recent api search.

{% Label %}sw-omnibox.js:{% endLabel %}

```js
...
const URL_CHROME_EXTENSIONS_DOC =
  'https://developer.chrome.com/docs/extensions/reference/';
const NUMBER_OF_PREVIOUS_SEARCHES = 4;

// Display the suggestions after user starts typing
chrome.omnibox.onInputChanged.addListener(async (input, suggest) => {
  await chrome.omnibox.setDefaultSuggestion({
    description: 'Enter a Chrome API or choose from past searches'
  });
  const { apiSuggestions } = await chrome.storage.local.get('apiSuggestions');
  const suggestions = apiSuggestions.map((api) => {
    return { content: api, description: `Open chrome.${api} API` };
  });
  suggest(suggestions);
});

// Open the reference page of the chosen API
chrome.omnibox.onInputEntered.addListener((input) => {
  chrome.tabs.create({ url: URL_CHROME_EXTENSIONS_DOC + input });
  // Save the latest keyword
  updateHistory(input);
});

async function updateHistory(input) {
  const { apiSuggestions } = await chrome.storage.local.get('apiSuggestions');
  apiSuggestions.unshift(input);
  apiSuggestions.splice(NUMBER_OF_PREVIOUS_SEARCHES);
  await chrome.storage.local.set({ apiSuggestions });
}
```

{% Aside 'important' %}

Extension service workers have access to both web APIs and Chrome APIs, with a few exceptions.
For a deep dive, see [Service Workers...](tbd) 

{% endAside %}

### Step 6: Set up a recurring event {: #step-6 }

The `setTimeout()` or `setInterval()` methods are commonly used to perform delayed or periodic
tasks. However, these APIs can fail because the scheduler will cancel the timers when the service
worker is terminated. Instead, extensions can use the [`chrome.alarms`][api-alarms] API. 

To use the Alarms API, request the `"alarms"` permission in the manifest. The extension also needs to request [host permission][doc-host-perm] to be able to fetch the extension tips from a remote hosted location:

{% Label %}manifest.json:{% endLabel %}

```json/2/3
{
  ...
  "permissions": ["storage", "alarms"],
  "permissions": ["storage"],
  "host_permissions": ["https://extension-tips.glitch.me/*"],
}
```

The following code sets up an alarm once a day to fetch the daily tip and save it to
[`chrome.storage.local()`][api-storage]:

{% Label %}sw-tips.js:{% endLabel %}

```js
// Fetch tip & save in storage
const updateTip = async () => {
  const response = await fetch('https://extension-tips.glitch.me/tips.json');
  const tips = await response.json();
  const randomIndex = Math.floor(Math.random() * tips.length);
  await chrome.storage.local.set({ tip: tips[randomIndex] });
};

// Create a daily alarm and retrieves the first tip when extension is installed.
chrome.runtime.onInstalled.addListener(({ reason }) => {
  if (reason === 'install') {
    chrome.alarms.create({ delayInMinutes: 1, periodInMinutes: 1440 });
    updateTip();
  }
});

// Update tip once a the day
chrome.alarms.onAlarm.addListener(updateTip);

```

{% Aside %}

All [Chrome API][doc-apis] event listeners and methods restart the service worker 30 second termination timer. See [Extension service worker lifecycle](tdb) to learn more.

{% endAside %}

### Step 7: Communicate with other contexts {: #step-7 }

[Content scripts][doc-content] communicate with the rest of the extension through [message passing][doc-messages]. In this example, the content script will request the tip of the day from the service worker. 

First, declare the content script in the manifest and add the match pattern corresponding to the [Chrome API][doc-apis] reference documentation.

{% Label %}manifest.json:{% endLabel %}

```json
{
  ...
  "content_scripts": [
    {
      "matches": ["https://developer.chrome.com/docs/extensions/reference/*"],
      "js": ["content.js"]
    }
  ]
}

```

Create a new content file. The following code generates a button that will open the tip popover. It also sends a message to the service worker requesting the extension tip.

{% Label %}content.js:{% endLabel %}

```js
(async () => {
  const nav = document.querySelector('.navigation-rail__links');

  const { tip } = await chrome.runtime.sendMessage({ greeting: 'tip' });

  const tipWidget = createDomElement(`
    <button class="navigation-rail__link" popovertarget="tip-popover" popovertargetaction="show" style="padding: 0; border: none; background: none;>
      <div class="navigation-rail__icon">
        <svg class="icon" xmlns="http://www.w3.org/2000/svg" aria-hidden="true" width="24" height="24" viewBox="0 0 24 24" fill="none"> 
        <path d='M15 16H9M14.5 9C14.5 7.61929 13.3807 6.5 12 6.5M6 9C6 11.2208 7.2066 13.1599 9 14.1973V18.5C9 19.8807 10.1193 21 11.5 21H12.5C13.8807 21 15 19.8807 15 18.5V14.1973C16.7934 13.1599 18 11.2208 18 9C18 5.68629 15.3137 3 12 3C8.68629 3 6 5.68629 6 9Z'"></path>
        </svg>
      </div>
      <span>Tip</span> 
    </button>
  `);

  const popover = createDomElement(
    `<div id='tip-popover' popover>${tip}</div>`
  );

  document.body.append(popover);
  nav.append(tipWidget);
})();

function createDomElement(html) {
  const dom = new DOMParser().parseFromString(html, 'text/html');
  return dom.body.firstElementChild;
}
```


{% Details %}
{% DetailsSummary %}
💡 **Interesting JavaScript used in this code**
{% endDetailsSummary %}

TBD

- DomParser
- Popover API
- SVG element

{% endDetails %}

The final step is to add a message handler to our service worker that replies the daily tip when requested from the content script. 

{% Label %}sw-api.js:{% endLabel %}

```js
...
// Send tip to content script via messaging
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.greeting === 'tip') {
    chrome.storage.local.get('tip').then(sendResponse);
    return true;
  }
});
```

## Test that it works {: #try-out }

Verify that the file structure of your project looks like the following: 

{% Img src="image/BhuKGJaIeLNPW9ehns59NfwqKxF2/bnfc65cdTGctM4A15DD9.png", alt="The contents of the extension folder: manifest.json, service-worker.js, sw-omnibox.js, sw-tips.js,
content.js, and icons.", width="400", height="538" %}

### Load your extension locally {: #locally }

To load an unpacked extension in developer mode, follow the steps in [Development
Basics][doc-dev-basics-unpacked].

### Open a reference page {: #open-api }

1. Enter the keyword "api" in the browser address bar.
1. Press "tab" or "space".
1. Enter the complete name of the API.
   - OR choose from a list of past searches
1. A new page will open to the Chrome API reference page.

It should look like this:

<figure>
{% Img src="image/BhuKGJaIeLNPW9ehns59NfwqKxF2/tKsdFmAYFGApMRF47Nlp.gif", alt="Quick API Reference opening the runtime api reference", width="500", height="306", class="screenshot" %}
  <figcaption>
  Quick API extension opening the Runtime API.
  </figcaption>
</figure>

### Open the tip of the day {: #open-tip }

Click the Tip button located on the navigation bar to open the extension tip.

<figure>
{% Img src="image/BhuKGJaIeLNPW9ehns59NfwqKxF2/GqjdrVtuA0zt87l3QIn9.gif", alt="Open daily tip in ", width="500", height="593" %}
  <figcaption>
  Quick API extension opening the Runtime API.
  </figcaption>
</figure>


## 🎯 Potential enhancements {: #challenge }

Based on what you’ve learned today, try to accomplish any of the following:

- Add a CSS stylesheet to the content script.
- TBD
- TBD

## Keep building! {: #continue }

Congratulations on finishing this tutorial 🎉. Continue leveling up your skills by completing other
tutorials on this series:

| Extension                        | What you will learn                                            |
|----------------------------------|----------------------------------------------------------------|
| [Reading time][tut-reading-time] | To insert an element on a specific set of pages automatically. |
| [Tabs Manager][tut-tabs-manager] | To create a popup that manages browser tabs.                   |

## Continue exploring

To continue your extension service worker learning path, we recommend exploring the following articles:

- TBD
- TBD
- TBD

[api-scripting]: /docs/extensions/reference/scripting/
[api-storage]: /docs/extensions/reference/storage
[api-alarms]: /docs/extensions/reference/alarms
[api-omnibox]: /docs/extensions/reference/omnibox
[doc-apis]: /docs/extensions/reference
[doc-dev-basics-unpacked]: /docs/extensions/mv3/getstarted/development-basics#load-unpacked
[doc-dev-basics]: /docs/extensions/mv3/getstarted/development-basics
[doc-devguide]: /docs/extensions/mv3/devguide/
[doc-ext-101]: /docs/extensions/mv3/getstarted/extensions-101/
[doc-manifest]: /docs/extensions/mv3/manifest/
[doc-perms-warning]: /docs/extensions/mv3/permission_warnings/#required_permissions
[doc-sw]: /docs/extensions/mv3/service_workers/
[doc-content]: /docs/extensions/mv3/content_scripts/
[doc-messages]: /docs/extensions/mv3/messaging
[doc-welcome]: /docs/extensions/mv3/
[github-focus-mode-icons]: https://github.com/GoogleChrome/chrome-extensions-samples/tree/main/functional-samples/tutorial.focus-mode/images
[github-open-api]: https://github.com/GoogleChrome/chrome-extensions-samples/tree/main/functional-samples/
[mdn-es-module]: https://web.dev/es-modules-in-sw/
[runtime-oninstalled]: /docs/extensions/reference/runtime#event-onInstalled
[tut-reading-time-step1]: /docs/extensions/mv3/getstarted/tut-reading-time#step-1
[tut-reading-time-step2]: /docs/extensions/mv3/getstarted/tut-reading-time#step-2
[tut-reading-time]: /docs/extensions/mv3/getstarted/tut-reading-time
[tut-tabs-manager]: /docs/extensions/mv3/getstarted/tut-tabs-manager
[mdn-local-storage]: https://developer.mozilla.org/docs/Web/API/Window/localStorage
[doc-host-perm]: /docs/extensions/mv3/match_patterns/
[mdn-window]: https://developer.mozilla.org/docs/Web/API/Window
[manifest-min-version]: /docs/extensions/mv3/manifest/minimum_chrome_version/#enforcement
[crxjs-vite]: https://crxjs.dev/vite-plugin