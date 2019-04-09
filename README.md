[![npm version](https://badge.fury.io/js/penpal.svg)](https://badge.fury.io/js/penpal)

[![Build Status](https://saucelabs.com/browser-matrix/Aaronius9erPenpalMaster.svg)](https://saucelabs.com/u/Aaronius9erPenpalMaster)

# Penpal

Penpal is a promise-based library for securely communicating with iframes via postMessage. The parent window can call methods exposed by iframes, pass arguments, and receive a return value. Similarly, iframes can call methods exposed by the parent window, pass arguments, and receive a return value. Easy peasy.

This library has no dependencies.

## Installation

### Using npm

Preferably, you'll be able to use Penpal from npm with a bundler like [Webpack](https://webpack.github.io/), [Rollup](https://rollupjs.org), or [Parcel](https://parceljs.org/). If you use npm for client package management, you can install Penpal with:

`npm install penpal --save`

### Using a CDN

If you don't want to use npm to manage client packages, Penpal also provides a UMD distribution in a `dist` folder which is hosted on a CDN:

`<script src="https://unpkg.com/penpal/dist/penpal.min.js"></script>`

Penpal will then be installed on `window.Penpal`. `window.Penpal` will contain the following properties:

```
Penpal.ERR_CONNECTION_DESTROYED
Penpal.ERR_CONNECTION_TIMEOUT
Penpal.ERR_NOT_IN_IFRAME
Penpal.ERR_IFRAME_ALREADY_ATTACHED_TO_DOM
Penpal.connectToChild
Penpal.connectToParent
```

Usage is similar to if you were using a bundler, which is documented below, but instead of importing each module, you would access it on the `Penpal` global instead.

## Usage

### Parent Window

```javascript
import connectToChild from 'penpal/lib/connectToChild';

const connection = connectToChild({
  // URL of page to load into iframe.
  url: 'http://example.com/iframe.html',
  // Container to which the iframe should be appended.
  appendTo: document.getElementById('iframeContainer'),
  // Methods parent is exposing to child
  methods: {
    add(num1, num2) {
      return num1 + num2;
    }
  }
});

connection.promise.then(child => {
  child.multiply(2, 6).then(total => console.log(total));
  child.divide(12, 4).then(total => console.log(total));
});
```

### Child Iframe

```javascript
import connectToParent from 'penpal/lib/connectToParent';

const connection = connectToParent({
  // Methods child is exposing to parent
  methods: {
    multiply(num1, num2) {
      return num1 * num2;
    },
    divide(num1, num2) {
      // Return a promise if the value being returned requires asynchronous processing.
      return new Promise(resolve => {
        setTimeout(() => {
          resolve(num1 / num2);
        }, 1000);
      });
    }
  }
});

connection.promise.then(parent => {
  parent.add(3, 1).then(total => console.log(total));
});
```

## API

### `connectToChild(options:Object) => Object`

#### Parameters

`options.url` (required) The URL of the webpage that should be loaded into the iframe that Penpal will create. A relative path is also supported.

`options.appendTo` (optional) The element to which the created iframe should be appended. If not provided, the iframe will be appended to `document.body`.

`options.iframe` (optional) The iframe element that Penpal should use instead of creating an iframe element itself. This iframe element must not be already appended to the DOM; it will be appended by Penpal. This option is useful if you need to set properties on the iframe element before it is appended to the DOM (for example, if you need to set the `sandbox` property). Note that the `src` property of the iframe will be set by Penpal using the `options.url` value, even if `src` has been set previously.

`options.methods` (optional) An object containing methods which should be exposed for the child iframe to call. The keys of the object are the method names and the values are the functions. If a function requires asynchronous processing to determine its return value, make the function immediately return a promise and resolve the promise once the value has been determined.

`options.timeout` (optional) The amount of time, in milliseconds, Penpal should wait for the child to respond before rejecting the connection promise. There is no timeout by default.

`options.debug` (optional) Enables or disables debug logging. Debug logging is disabled by default.

#### Return value

The return value of `connectToChild` is a `connection` object with the following properties:

`connection.promise` A promise which will be resolved once communication has been established. The promise will be resolved with an object containing the methods which the child has exposed. Note that these aren't actual memory references to the methods the child exposed, but instead proxy methods Penpal has created with the same names and signatures. When one of these methods is called, Penpal will immediately return a promise and then go to work sending a message to the child, calling the actual method within the child with the arguments you have passed, and then sending the return value back to the parent. The promise you received will then be resolved with the return value.

`connection.destroy` A method that, when called, will remove the iframe element from the DOM and disconnect any messaging channels. You may call this even before a connection has been established.

`connection.iframe` The child iframe element. The iframe will have already been appended as a child to the element defined in `options.appendTo`, but a reference to the iframe is provided in case you need to add CSS classes, etc.

### `connectToParent([options:Object]) => Object`

#### Parameters

`options.parentOrigin` (optional) The origin of the parent window which your iframe will be communicating with. If this is not provided, communication will not be restricted to any particular parent origin resulting in any webpage being able to load your webpage into an iframe and communicate with it.

`options.methods` (optional) An object containing methods which should be exposed for the parent window to call. The keys of the object are the method names and the values are the functions. If a function requires asynchronous processing to determine its return value, make the function immediately return a promise and resolve the promise once the value has been determined.

`options.timeout` (optional) The amount of time, in milliseconds, Penpal should wait for the parent to respond before rejecting the connection promise. There is no timeout by default.

`options.debug` (optional) Enables or disables debug logging. Debug logging is disabled by default.

#### Return value

The return value of `connectToParent` is a `connection` object with the following property:

`connection.promise` A promise which will be resolved once communication has been established. The promise will be resolved with an object containing the methods which the parent has exposed. Note that these aren't actual memory references to the methods the parent exposed, but instead proxy methods Penpal has created with the same names and signatures. When one of these methods is called, Penpal will immediately return a promise and then go to work sending a message to the parent, calling the actual method within the parent with the arguments you have passed, and then sending the return value back to the child. The promise you received will then be resolved with the return value.

`connection.destroy` A method that, when called, will disconnect any messaging channels. You may call this even before a connection has been established.

## Reconnection

If the child iframe attempts to reconnect with the parent, the parent will accept the new connection. This could happen, for example, if a user refreshes the child iframe or navigates within the iframe to a different page that also uses Penpal. In this case, the `child` object the parent received when the initial connection was established will be updated with the new methods provided by the child iframe.

NOTE: Currently there is no API to notify consumers of a reconnection. If this is important for you, please file an issue and explain why it would be beneficial to you.

## Errors

Penpal will throw (or reject promises with) errors in certain situations. Each error will have a `code` property which may be used for programmatic decisioning (e.g., do something if the error was due to a connection timing out) along with a `message` describing the problem. Errors may be thrown with the following codes:

- `ConnectionDestroyed`
  - `connection.promise` will be rejected with this error if the connection is destroyed (by calling `connection.destroy()`) while Penpal is attempting to establish the connection.
  - This error will be thrown when attempting to call a method on `child` or `parent` objects and the connection was previously destroyed.
- `ConnectionTimeout`
  - `connection.promise` will be rejected with this error after the `timeout` duration has elapsed and a connection has not been established.
- `NotInIframe`
  - This error will be thrown when attempting to call `Penpal.connectToParent()` from outside of an iframe context.
- `IframeAlreadyAttachedToDom`
  - This error will be thrown when an iframe already attached to the DOM is passed to `Penpal.connectToChild()`.

For your convenience, these error codes are exported as constants that can be imported as follows:

```
import {
  ERR_CONNECTION_DESTROYED,
  ERR_CONNECTION_TIMEOUT,
  ERR_NOT_IN_IFRAME,
  ERR_IFRAME_ALREADY_ATTACHED_TO_DOM
} from 'penpal/lib/errorCodes';
```

## Security Note

Penpal does not set the [`sandbox` property](https://www.html5rocks.com/en/tutorials/security/sandboxed-iframes/) on the iframe element it creates. If you would like to sandbox the iframe, you must, in the parent, create the iframe element, set its `sandbox` property, then pass the iframe to the `connectToChild` method. Failing to set the `sandbox` property on the iframe prior to Penpal adding the iframe to the DOM [can fail to properly enforce sandbox security](https://bugzilla.mozilla.org/show_bug.cgi?id=1522702). The following example demonstrates setting the `sandbox` property on the iframe from the parent window:

```javascript
import Penpal from 'penpal';

const iframe = document.createElement('iframe');
iframe.sandbox = 'allow-scripts';

const connection = Penpal.connectToChild({
  // URL of page to load into iframe.
  url: 'http://example.com/iframe.html',
  // Container to which the iframe should be appended.
  appendTo: document.getElementById('iframeContainer'),
  // The iframe element to use
  iframe: iframe,
  // Methods parent is exposing to child
  methods: {
    add(num1, num2) {
      return num1 + num2;
    }
  }
});

connection.promise.then(child => {
  child.multiply(2, 6).then(total => console.log(total));
  child.divide(12, 4).then(total => console.log(total));
});
```

## Supported Browsers

Penpal is designed to run successfully on the most recent versions of Edge, Chrome, Firefox, and Safari.

## Inspiration

This library is inspired by:

- [Postmate](https://github.com/dollarshaveclub/postmate)
- [JSChannel](https://github.com/mozilla/jschannel)

## License

MIT
