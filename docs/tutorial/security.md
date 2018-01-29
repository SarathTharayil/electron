# Security, Native Capabilities, and Your Responsibility

As web developers, we usually enjoy the strong security net of the browser - the
risks associated with the code we write are relatively small. Our websites are
granted limited powers in a sandbox, and we trust that our users enjoy a browser
built by a large team of engineers that is able to quickly respond to newly
discovered security threats.

When working with Electron, it is important to understand that Electron is not
a web browser. It allows you to build feature-rich desktop applications with
familiar web technologies, but your code wields much greater power. JavaScript
can access the filesystem, user shell, and more. This allows you to build
high quality native applications, but the inherent security risks scale with the
additional powers granted to your code.

With that in mind, be aware that displaying arbitrary content from untrusted
sources poses a severe security risk that Electron is not intended to handle.
In fact, the most popular Electron apps (Atom, Slack, Visual Studio Code, etc)
display primarily local content (or trusted, secure remote content without Node
integration) – if your application executes code from an online source, it is
your responsibility to ensure that the code is not malicious.

## Reporting Security Issues

For information on how to properly disclose an Electron vulnerability,
see [SECURITY.md](https://github.com/electron/electron/tree/master/SECURITY.md)

## Chromium Security Issues and Upgrades

While Electron strives to support new versions of Chromium as soon as possible,
developers should be aware that upgrading is a serious undertaking - involving
hand-editing dozens or even hundreds of files. Given the resources and
contributions available today, Electron will often not be on the very latest
version of Chromium, lagging behind by either days or weeks.

We feel that our current system of updating the Chromium component strikes an
appropriate balance between the resources we have available and the needs of the
majority of applications built on top of the framework. We definitely are
interested in hearing more about specific use cases from the people that build
things on top of Electron. Pull requests and contributions supporting this
effort are always very welcome.

## Ignoring Above Advice

A security issue exists whenever you receive code from a remote destination and
execute it locally. As an example, consider a remote website being displayed
inside a `BrowserWindow`. If an attacker somehow manages to change said content
(either by attacking the source directly, or by sitting between your app and
the actual destination), they will be able to execute native code on the user's
machine.

> :warning: Under no circumstances should you load and execute remote code with
Node.js integration enabled. Instead, use only local files (packaged together with
your application) to execute Node.js code. To display remote content, use the
`webview` tag and make sure to disable the `nodeIntegration`.

#### Checklist: Security Recommendations

This is not bulletproof, but at the least, you should attempt the following:

* [Only display secure (https) content](#only-display-secure-content)
* [Disable the Node integration in all renderers that display remote content](#disable-node-integration-for-remote-content)
* [Enable context isolation in all renderers that display remote content](#enable-context-isolation-for-remote-content)
* [Use `ses.setPermissionRequestHandler()` in all sessions that load remote content](#handle-session-permission-requests-from-remote-content)
* [Do not disable `webSecurity`](#do-not-disable-websecurity)
* [Define a `Content-Security-Policy`](#define-a-content-security-policy)
, and use restrictive rules (i.e. `script-src 'self'`)
* [Override and disable `eval`](#override-and-disable)
, which allows strings to be executed as code.
* [Do not set `allowRunningInsecureContent` to `true`](#do-not-set-allowRunningInsecureContent-to-true)
* [Do not enable experimental features](#do-not-enable-enable-experimental-features)
* [Do not use `blinkFeatures`](#do-not-use-blinkfeatures)
* WebViews: Do not add the `nodeintegration` attribute.
* WebViews: Do not use `allowpopups`
* WebViews: Do not use `insertCSS` or `executeJavaScript` with remote CSS/JS.
* [WebViews: Verify the options and params of all `<webview>` tags](#verify-webview-options-before-creation)

## Only Display Secure Content

Any resources not included with your application should be loaded using a secure
protocol like `HTTPS`.

### Why?

`HTTPS` has three main benefits:

1) It authenticates the remote server, ensuring that the host is actually who it
   claims to be. When loading a resource from an `HTTPS` host, it prevents an
   attacker from impersonating that host, thus ensuring that the computer your
   app's users are connecting to is actually the host you wanted them to connect
   to.
2) It ensures data integrity, asserting that the data was not modified while in
   transit between your application and the host.
3) It encryps the traffic between your user and the destination host, making it
   more difficult to eavesdropping on the information sent between your app and
   the host.

### How?

```js
// Bad
browserWindow.loadURL('http://my-website.com')

// Good
browserWindow.loadURL('https://my-website.com')
```

```html
<!-- Bad -->
<script crossorigin src="http://cdn.com/react.js"></script>
<link rel="stylesheet" href="http://cdn.com/style.css">

<!-- Good -->
<script crossorigin src="https://cdn.com/react.js"></script>
<link rel="stylesheet" href="https://cdn.com/style.css">
```

## Disable Node Integration for Remote Content

It is paramount that you disable Node integration in any renderer (`BrowserWindow`,
`BrowserView`, or `WebView`) that loads remote content. The goal of disabling Node
integration is to limit the powers you grant to remote content, thus making it
dramatically more difficult for an attacker to harm your users should they gain
the ability to execute JavaScript on your website.

Disabling Node integration does not mean that you cannot grant additional powers
to the website you are loading. If you are opening a `BrowserWindow` pointed at
`https://my-website.com`, the goal is to give that website exactly the abilities
it needs, but no more.

### Why?

A cross-site-scripting (XSS) attack becomes dramatically more dangerous if an
attacker can jump out of the renderer process and execute code on the user's
computer. Cross-site-scripting attacks are fairly common - and while an issue,
their power is usually limited to messing with the website that they are executed
on. However, in a renderer process with Node.js integration enabled, an XSS attack
becomes a whole different class of attack: A so-called "Remote Code Execution"
(RCE) attack. Disabling Node.js integration limits the power of successful XSS
attacks.

### How?

```js
// Bad
const mainWindow = new BrowserWindow()
mainWindow.loadURL('https://my-website.com')

// Good
const mainWindow = new BrowserWindow({
  webPreferences: {
    nodeIntegration: false,
    preload: './preload.js'
  }
})

mainWindow.loadURL('https://my-website.com')
```

When disabling Node integration, you can still expose APIs to your
website that do consume Node.js modules or features. Preload scripts continue to
have access to `require` and other Node.js features, allowing developers to expose
a custom API to remotely loaded content.

In the following example preload script, the later loaded website will have access
to a `window.readConfig()` method, but no Node.js features.

```js
const { readFileSync } = require('fs')

window.readConfig = function () {
  const data = readFileSync('./config.json')
  return data;
}
```

## Enable Context Isolation for Remote Content

## Handle Session Permission Requests From Remote Content

You may have seen permission requests while using Chrome: They pop up whenever
the website attempts to use a feature that the user has to manually approve (
like notifications).

// Todo: What are the events?

### Why?

By default, Electron will automatically approve all permission requests unless
the developer has manually configured a custom handler. While a solid default,
security-concious developers might want to assume the very opposite.

### How?

```js
const { session } = require('electron')

session
  .fromPartition('some-partition')
  .setPermissionRequestHandler((webContents, permission, callback) => {
    const url = webContents.getURL()

    if (permission === 'notifications') {
      // Approves the permissions request
      callback(true)
    }

    if (!url.startsWith('https://my-website.com')) {
      // Denies the permissions request
      return callback(false)
    }
  })
```

## Define a Content Security Policy

## Override and Disable `eval`

## Do Not Set `allowRunningInsecureContent` to `true`

By default, Electron will now allow websites loaded over `HTTPS` to load and
execute scripts, CSS, or plugins from insecure sources (`HTTP`). Setting the
property `allowRunningInsecureContent` to `true` disables that protection.

Loading the initial HTML of a website over `HTTPS` and attempting to load
subsequent resources via `HTTP` is also known as "mixed content".

### Why?

See the section on [only displaying secure content](#only-display-secure-content)
for more details, but simply put, loading content over `HTTPS` assures the
authenticity and integrity of the loaded resources while encrypting the traffic
itself.

### How?

```js
// Bad
const mainWindow = new BrowserWindow({
  webPreferences: {
    allowRunningInsecureContent: true
  }
})

// Good
const mainWindow = new BrowserWindow({})
```

## Do Not Enable Experimental Features

Advanced users of Electron can enable experimental Chromium features using the
`experimentalFeatures` and `experimentalCanvasFeatures` properties.

### Why?

Experimental features are, as the name suggests, experimental and have not been
enabled for all Chromium users. Futhermore, their impact on Electron as a whole
has likely not been tested.

Legitimate use cases exist, but unless you know what you are doing, you should
not enable this property.

### How?

```js
// Bad
const mainWindow = new BrowserWindow({
  webPreferences: {
    experimentalFeatures: true
  }
})

// Good
const mainWindow = new BrowserWindow({})
```

## Do Not Disable WebSecurity

You may have already guessed that disabling the `webSecurity` property on a
renderer process (`BrowserView`, `BrowserWindow`, `WebView`) disables crucial
security features.

Legitimate use cases for this property exist in testing cases, but generally
speaking, `webSecurity` should never be disabled in any production application.

### Why?

Disbling `webSecurity` will disable the same-origin policy as well as implicitly
setting the `allowRunningInsecureContent` property to `true`. In other words,
it allows the execution of insecure code from different domains.

// Todo: Explain same-origin policy

### How?
```js
// Bad
const mainWindow = new BrowserWindow({
  webPreferences: {
    webSecurity: false
  }
})

// Good
const mainWindow = new BrowserWindow()
```

```html
<!-- Bad -->
<webview disablewebsecurity src="page.html"></webview>

<!-- Good -->
<webview src="page.html"></webview>
```

## Do Not Use `blinkFeatures`
Blink is the name of the rendering engine behind Chromium.

// Todo: Way more!

### How?
```js
// Bad
const mainWindow = new BrowserWindow({
  webPreferences: {
    blinkFeatures: []
  }
})

// Good
const mainWindow = new BrowserWindow()
```

## Verify WebView Options Before Creation

A WebView created in a renderer process that does not have Node.js integration
enabled will not be able to enable integration itself. However, a WebView will
always create an independent renderer process with its own `webPreferences`.

It is a good idea to control the creation of new `WebViews` from the main process
and to verify that their webPreferences do not disable security features.

### Why?

Since WebViews live in the DOM, they can be created by a script running on your
website even if Node integration is otherwise disabled.

Electron enables developers to disable various security features that control
a renderer process. In most cases, developers do not need to disable any of those
features - and you should therefore not allow different configurations for newly
created `<WebView>` tags.

### How?

Before a `<WebView>` tag is attached, Electron will fire the
`will-attach-webview` event on the hosting `webContents`. Use the event to
prevent the creation of WebViews with possibly insecure options.

```js
app.on('web-contents-created', (event, contents) => {
  contents.on('will-attach-webview', (event, webPreferences, params) => {
    // Strip away preload scripts if unused or verify their location is legitimate
    delete webPreferences.preload
    delete webPreferences.preloadURL

    // Disable node integration
    webPreferences.nodeIntegration = false

    // Verify URL being loaded
    if (!params.src.startsWith('https://yourapp.com/')) {
      event.preventDefault()
    }
  })
})
```

Again, this list merely minimizes the risk, it does not remove it. If your goal
is to display a website, a browser will be a more secure option.
