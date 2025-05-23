# Product Specification: WebExtension Emulation Library (WEEL) version 1.0.0

## Product Name

WebExtension Emulation Library (WEEL, pronounced “wheel”)

## Version

1.0.0

## Product Goal

To provide a JavaScript library that accurately emulates the runtime environment of a WebExtension within a standard web page context.
This enables developers to test, debug, and prototype WebExtension features directly in the browser's developer tools without the need for repeated installation and uninstallation of the extension.

## Target Audience:

- WebExtension Developers
- Front-end Developers working with browser APIs
- Testers and QA engineers for WebExtensions
- Educators teaching WebExtension development

## Problem Statement

Developing and debugging WebExtensions can be cumbersome.
Each change often requires reloading or reinstalling the extension, and debugging often involves specific extension-debugging tools that can be less familiar than standard browser developer tools.
There's a need for a more streamlined development workflow that allows for quicker iteration and easier debugging.

## Solution Overview

WEEL will provide a JavaScript API that, when initialized, recreates key aspects of a WebExtension's runtime environment.
This includes:

- Mocking core `chrome.*` and `browser.*` APIs.
- Simulating messaging between different extension contexts (background scripts, content scripts, popups, options pages).
- Providing a mechanism to register and "run" simulated background scripts and content scripts.
- Offering configurable settings to mimic different browser environments (e.g. Chrome, Firefox).

## Key Features

- **Core API Emulation:**
  - Support global namespaces `chrome` and `browser` (the exposed globals depend on browser environment selection)
  - Start by emulating common WebExtensions APIs, including but not limited to:
    - `runtime` (e.g. `sendMessage`, `onMessage`, `connect`, `onConnect`, `getURL`, `getManifest`, `id`)
    - `tabs` (e.g. `query`, `sendMessage`, `executeScript`, `create`, `update`)
    - `storage` (`local`, `sync`, `session`, `managed`)
    - `notifications`
    - `alarms`
    - `contextMenus`
    - `i18n`
- **Context Simulation:**
  - Background Script: Ability to load and execute background JavaScript in a appropriate simulated execution context (persistent page, event page, or service worker).
    - Simulate termination of ephemeral contexts when idle (event page, service worker), with idle length configurable at initialization time.
  - Content Script: Ability to inject and execute JavaScript and CSS into a simulated tab and subframe contexts.
  - Popup/Options Page: Mechanism to register and interact with simulated popup and options page scripts.
- **Messaging System:**
  - Simulate `runtime.sendMessage` and `tabs.sendMessage` for communication between emulated contexts, including broadcasting 
  - Support for long-lived connections via cross-browser APIs such as `browser.runtime.connect` and browser-specific interfaces like `chrome.runtime.connectNative`.
  - Properly handle channel termination when the execution context of a given port terminates.
- **Storage Emulation:**
  - Provide a local storage mechanism that mimics the `local`, `sync`, `session`, and `managed` storage areas.
- **Event System:**
  - Emulate the `addListener`, `removeListener` and `hasListener` methods for WebExtension events.
  - Emulate dispatch of events to registered listeners.
- **Manifest Loading:**
  - Ability to load and parse a `manifest.json` file to configure the extension's environment.
  - Support for key manifest keys (e.g. `permissions`, `background`, `content_scripts`, `action`, `options`).
- **Configurability:**
  - Option to specify the target browser environment (Chrome, Firefox, Edge) to adjust API behavior nuances.
  - Ability to enable/disable specific API emulations.
  - Debug mode for verbose logging of emulated API calls.
- **Extensibility:**
  - Provide hooks for developers to extend or override default API emulations.
  - Allow for custom mock data to be injected into emulated APIs.

## Non-Goals

- Full emulation of all browser-specific WebExtension APIs (e.g. highly specialized APIs like `chrome.debugger` or `browser.devtools`).
- Running packed WebExtensions (`.zip`, `.crx`, `.xpi`) directly. This library focuses on emulating the runtime environment for experimentation and testing purposes.
- Performance benchmarking of WebExtensions.

## Technical Design (High-Level)

- **Core Module:**
  - A central `WEEL` object that manages the emulation.
  - Contains methods for initialization, loading manifest, and registering scripts.
  - Manages dynamic loading of API-specific sub-modules.
- **Common API Mocking Layer:**
  - Uses JavaScript Proxies or direct object property assignment to intercept and emulate `chrome` and `browser` API calls.
  - Each API will have a dedicated module for its emulation logic.
- **Browser Mocking Layer:**
  - Allows for registration of tailored mocks to more closely emulate the behavior of a given browser engine.
  - Each browser will have a dedicated directory that contains it’s modules for it’s emulation logic.
- **Messaging System:**
  - Utilize an internal Event Emitter or Message Channel to simulate inter-context communication.
- **Storage Layer:**
  - In-memory storage for default, with an option to persist to localStorage or IndexedDB.
- **Script Runner:**
  - Uses eval or Function constructor to execute provided script strings within the emulated context.

## API Design (Examples):

```js
// Initialization
//
const weel = new WEEL.EmulationEnvironment({
  browser: 'unset', // or 'firefox', 'safari', 'chrome'
  manifest: {
    // manifest.json properties ...
  }, 
  hostAccess: 'onClick', // or 'all', 'manual'
  permissions: {
    origins: [
      // list of 
    ],
    permissions: [
      // list of permissions grants
    ]
  },
  policies: {
    // enterprise policies that apply to this extension
  }
});

// Register a background script
weel.setBackgroundScript({
  path: 'background.js',
  code: `
    chrome.runtime.onInstalled.addListener(() => {
      console.log('Background script installed (emulated)');
    });

    chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
      if (message.action === 'getData') {
        sendResponse({ data: 'Hello from background!' });
      }
    });
  `
});


weel.setPermissions({

});

// Register a content script for a specific URL pattern
weel.registerContentScript('https://example.com/*', `
  chrome.runtime.sendMessage({ action: 'getData' }, (response) => {
    console.log('Received from background:', response.data);
  });
`);

// Simulate a tab interaction (e.g. sending a message from a content script)
weel.simulateTab({ id: 1, url: 'https://example.com/page' });
// Then you can trigger content script execution or send messages from this simulated tab context
weel.sendMessageToContentScript(1, { action: 'simulateClick' });

// Access emulated storage
weel.getEmulatedStorage().local.set({ myKey: 'myValue' });
```

## User Interface (N/A)

This is a headless JavaScript library with no direct UI.
All interactions will be via the library's API.
In the future we may consider creating a UI library to visualize browser behaviors.

## Integration Points

- Can be used in development servers (e.g. Webpack Dev Server, Vite) to provide a live emulation environment.
- Can be embedded directly into a web page for interactive prototyping.
- Exposes API to enable a REPL-like experience, allowing developers to iteratively test event handlers.

## Performance Requirements

- Minimal overhead when emulating APIs.
- Fast initialization time.
- Emulation should not significantly degrade browser performance.
- Must be able to load, unload, and replace registers event handlers.

## Security Considerations

- The library will execute user-provided script strings (`registerBackgroundScript`, `registerContentScript`).
  Developers should be aware of the security implications when using untrusted code.
- Since the library runs in the context of the host page, it inherits the host page's security permissions.
  Only limited attempts are made to provide an isolated extension security sandbox.
- User-provided scripts are executed in an isolated JavaScript context (an iframe or a Worker as appropriate) in order to more closely emulate extension behavior.

## Testing Strategy

- Unit Tests: Comprehensive unit tests for each emulated API, covering various inputs and expected outputs.
- Integration Tests: Tests to ensure correct communication and behavior between different emulated contexts (background, content, popup).
- Conformance Tests: Tests against actual WebExtension API specifications to ensure high fidelity emulation.
- Browser Compatibility Tests: Test across major browsers (Chrome, Firefox, Edge) to ensure consistent behavior.

## Future Considerations (Potential Enhancements)

- Support for more advanced manifest keys (e.g. declarativeNetRequest).
- Visual UI for inspecting emulated state (storage, active listeners).
- Integration with browser developer tools for a more seamless debugging experience.
- Support for more specialized APIs (e.g. chrome.debugger with limited functionality).
- Record and replay functionality for API calls.
- Virtualize the file system to more closely emulate behaviors related to changing files on disk during development, the extension update flow, and working with statically bundled extension resources.
- Ability to integrate into testing frameworks (e.g. Jest, Mocha, Cypress).
- Allow storage areas to persist data across executions (configurable).

## Open Questions

- How to handle complex API interactions like `browser.webRequest` and `filters and blocking requests?
  (Initial implementation will focus on event listeners).
- What level of fidelity is required for different `chrome.tabs` methods, especially those that involve actual browser UI manipulation?
  (Initial focus on data manipulation and messaging).
- How to best manage the lifecycle of emulated contexts (e.g. simulating tab closing, extension unloading)?

## Success Metrics

- Developer adoption and positive feedback.
- Reduction in extension development and debugging time.
- High test coverage of the emulation library.
- Accuracy of API emulation (measured by conformance tests).
- Ease of integration into existing development workflows.
- Used by MDN WebExtensions documentation to let users experiment with extensions APIs.
