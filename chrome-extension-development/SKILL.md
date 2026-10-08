---
name: chrome-extension-development
description: "Expert guidelines for Chrome extension development with Manifest V3, covering security, performance, and best practices. Use when building browser extensions, creating popup UIs, implementing content scripts, working with Chrome APIs, managing extension permissions, or publishing to Chrome Web Store."
metadata:
  maintainer: Mindrally
  source: https://github.com/Mindrally/skills
---

# Chrome Extension Development

This skill provides expert-level guidance for Chrome extension development, covering JavaScript/TypeScript, browser extension APIs, and modern web development practices.

## Workflow: Building a Chrome Extension from Scratch

1. **Initialize the project** — Create the directory structure with `manifest.json`, background service worker, content scripts, and popup files.
2. **Configure the manifest** — Define permissions, content script matches, service worker registration, and action settings in Manifest V3 format.
3. **Implement the background service worker** — Set up event listeners for extension lifecycle, messaging, and alarms using the `chrome.*` API.
4. **Build content scripts** — Write scripts that interact with web page DOM, communicate with the background worker via `chrome.runtime.sendMessage`, and respect CSP.
5. **Create the popup UI** — Design the popup HTML/CSS and wire up interactivity with the background and content scripts.
6. **Add storage and state management** — Use `chrome.storage.local` or `chrome.storage.sync` to persist user settings and extension state.
7. **Test and debug** — Load the extension unpacked via `chrome://extensions`, use Chrome DevTools to inspect the service worker and content scripts, and run unit tests.
8. **Package and publish** — Prepare store assets (icons, screenshots, description), create a privacy policy, and submit to the Chrome Web Store.

## Example: Minimal Manifest V3 Configuration

```json
{
  "manifest_version": 3,
  "name": "My Extension",
  "version": "1.0.0",
  "description": "A sample Chrome extension using Manifest V3",
  "permissions": ["storage", "activeTab"],
  "action": {
    "default_popup": "popup/popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  "background": {
    "service_worker": "background/service-worker.js",
    "type": "module"
  },
  "content_scripts": [
    {
      "matches": ["https://*.example.com/*"],
      "js": ["content/content-script.js"],
      "css": ["content/styles.css"]
    }
  ]
}
```

## Example: Background Service Worker with Messaging

```typescript
// background/service-worker.ts

// Listen for extension install or update
chrome.runtime.onInstalled.addListener((details) => {
  if (details.reason === 'install') {
    chrome.storage.local.set({ initialized: true, count: 0 });
    console.log('Extension installed');
  }
});

// Handle messages from content scripts or popup
chrome.runtime.onMessage.addListener(
  (message: { type: string; payload?: unknown }, sender, sendResponse) => {
    if (message.type === 'GET_COUNT') {
      chrome.storage.local.get('count', (result) => {
        sendResponse({ count: result.count ?? 0 });
      });
      return true; // keep message channel open for async response
    }

    if (message.type === 'INCREMENT') {
      chrome.storage.local.get('count', (result) => {
        const newCount = (result.count ?? 0) + 1;
        chrome.storage.local.set({ count: newCount }, () => {
          sendResponse({ count: newCount });
        });
      });
      return true;
    }
  }
);

// Schedule periodic tasks with chrome.alarms
chrome.alarms.create('sync-data', { periodInMinutes: 30 });
chrome.alarms.onAlarm.addListener((alarm) => {
  if (alarm.name === 'sync-data') {
    console.log('Running scheduled sync');
  }
});
```

## Code Style and Structure

- Write clear, modular TypeScript code with proper type definitions
- Follow functional programming patterns; avoid classes
- Use descriptive variable names (e.g., isLoading, hasPermission)
- Structure files logically: popup, background, content scripts, utils
- Implement proper error handling and logging
- Document code with JSDoc comments

## Architecture and Best Practices

- Strictly follow Manifest V3 specifications
- Divide responsibilities between background, content scripts and popup
- Configure permissions following the principle of least privilege
- Use modern build tools (webpack/vite) for development
- Implement proper version control and change management

## Chrome API Usage

- Use chrome.* APIs correctly (storage, tabs, runtime, etc.)
- Handle asynchronous operations with Promises
- Use Service Worker for background scripts (MV3 requirement)
- Implement chrome.alarms for scheduled tasks
- Use chrome.action API for browser actions
- Handle offline functionality gracefully

## Security and Privacy

- Implement Content Security Policy (CSP)
- Handle user data securely
- Prevent XSS and injection attacks
- Use secure messaging between components
- Handle cross-origin requests safely
- Implement secure data encryption
- Follow web_accessible_resources best practices

## Performance and Optimization

- Minimize resource usage and avoid memory leaks
- Optimize background script performance
- Implement proper caching mechanisms
- Handle asynchronous operations efficiently
- Monitor and optimize CPU/memory usage

## UI and User Experience

- Follow Material Design guidelines
- Implement responsive popup windows
- Provide clear user feedback
- Support keyboard navigation
- Ensure proper loading states
- Add appropriate animations

## Internationalization

- Use chrome.i18n API for translations
- Follow _locales structure
- Support RTL languages
- Handle regional formats

## Accessibility

- Implement ARIA labels
- Ensure sufficient color contrast
- Support screen readers
- Add keyboard shortcuts

## Testing and Debugging

- Use Chrome DevTools effectively
- Write unit and integration tests
- Test cross-browser compatibility
- Monitor performance metrics
- Handle error scenarios

## Publishing and Maintenance

- Prepare store listings and screenshots
- Write clear privacy policies
- Implement update mechanisms
- Handle user feedback
- Maintain documentation

## Follow Official Documentation

- Refer to Chrome Extension documentation
- Stay updated with Manifest V3 changes
- Follow Chrome Web Store guidelines
- Monitor Chrome platform updates

## Output Expectations

- Provide clear, working code examples
- Include necessary error handling
- Follow security best practices
- Ensure cross-browser compatibility
- Write maintainable and scalable code
