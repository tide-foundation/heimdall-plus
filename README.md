# TideCloak-js
OpenIDConnect and Tide-Protocol compatible client side JS library. This library was built on top of the existing [keycloak-js](https://www.keycloak.org/securing-apps/javascript-adapter) codebase, so all of keycloak-js' functions are available here, alongside Tide specific addons described below. 

## Prerequisites can be found [here.](https://docs.tidecloak.com/docs/EncryptDecrypt/SetupED)

## Network Flow of TideCloak-js Encryption
```mermaid
sequenceDiagram

box Blue Browser
participant TideCloak-js
participant Client
end

participant TideCloak
participant ORKs

Note over Client, TideCloak-js: User has already authenticated
Note over Client: Collect sensitive user information. <br> Assign tags to each piece of info
Client ->> TideCloak-js: Encrypt(data)
critical Check user has access roles for the info tags
Note over TideCloak-js: Check realm access roles in token
option Not all tags found in access roles
TideCloak-js --> Client: Error
end
Note over TideCloak-js: Encrypt each piece of data with its own ephermeral key
TideCloak-js ->> ORKs: Ephermal Keys, Tags, Encrypted data, Token
Note over ORKs: Check Token has all tags as part of access roles
ORKs ->> TideCloak-js: Signed emphermal keys
Note over TideCloak-js: Serialize ephermeral keys + encrypted data into TideSerializedField
TideCloak-js ->> Client: TideSerializedFields
```

## Initialization
> `npm install tidecloak-js`
```javascript
import TideCloak from "tidecloak-js"
import tcData from "/tidecloak.json";

const tidecloak = new TideCloak({
  url: tcData['auth-server-url'],
  realm: tcData['realm'],
  clientId: tcData['resource'],
  vendorId: tcData['vendorId'],
  homeOrkUrl: tcData['homeOrkUrl']
});
```
## Encryption
```javascript
// tidecloak.encrypt returns string[] where the list are the encrypted strings
// passed in the parameter object. Order returned is same order as what was passed.
const encrypted_dob = await tidecloak.encrypt([
  {
    "data": "03/04/2005",
    "tags": ["dob"]
  }
])[0];

// before testing the below code, make sure you've set up the respected roles
const multi_encrypted_addresses = await tidecloak.encrypt([
  {
    "data": "10 Smith Street",
    "tags": ["street"]
  },
  {
    "data": "Southport",
    "tags": ["suburb"]
  },
  {
    "data": "20 James Street - Burleigh Heads",
    "tags": ["street", "suburb"]
  }
]);
```
When encrypting or decrypting data, a user must have permission for all the tags attached to that data. For instance, if data is tagged "street", a user with the _tide_street.selfencrypt role can handle that data, but if data includes multiple tags (like "street" and "suburb"), the user must have both corresponding roles to access it.

## Decryption
```javascript

// tidecloak.decrypt returns string[] where the list are the decrypted strings
// passed in the parameter object. Order returned is same order as what was passed.
const decrypted_dob = await tidecloak.decrypt([
  {
    "encrypted": encrypted_dob, // from the encrypt code block above
    "tags": ["dob"]
  }
]);

// before testing the below code, make sure you've set up the respected roles
const decrypted_addresses = await tidecloak.decrypt([
  {
    "encrypted": multi_encrypted_addresses[0],
    "tags": ["street"]
  },
  {
    "encrypted": multi_encrypted_addresses[1],
    "tags": ["suburb"]
  },
  {
    "encrypted": multi_encrypted_addresses[2],
    "tags": ["street", "suburb"]
  }
]);
```

# Reference guide

## TideCloak server configuration

One important thing to consider about using client-side applications is that the client has to be a public client as there is no secure way to store client credentials in a client-side application. This consideration makes it very important to make sure the redirect URIs you have configured for the client are correct and as specific as possible.

To use the adapter, create a client for your application in the TideCloak Admin Console. Make the client public by toggling *Client authentication*  to *Off*  on the *Capability config* page.

You also need to configure `Valid Redirect URIs` and `Web Origins`. Be as specific as possible as failing to do so may result in a security vulnerability.

## Using the adapter

The following example shows how to initialize the adapter. Make sure that you replace the options passed to the `TideCloak` constructor with those of the client you have configured.

```javascript
import TideCloak from 'tidecloak-js';
import tcData from "/tidecloak.json";

const tidecloak = new TideCloak({
  url: tcData['auth-server-url'],
  realm: tcData['realm'],
  clientId: tcData['resource'],
  vendorId: tcData['vendorId'],
  homeOrkUrl: tcData['homeOrkUrl']
});

try {
    const authenticated = await tidecloak.init();
    if (authenticated) {
        console.log('User is authenticated');
    } else {
        console.log('User is not authenticated');
    }
} catch (error) {
    console.error('Failed to initialize adapter:', error);
}
```

To authenticate, you call the `login` function. Two options exist to make the adapter automatically authenticate. You can pass `login-required` or `check-sso` to the `init()` function.

* `login-required` authenticates the client if the user is logged in to TideCloak or displays the login page if the user is not logged in.
* `check-sso` only authenticates the client if the user is already logged in. If the user is not logged in, the browser is redirected back to the application and remains unauthenticated.

You can configure a _silent_ `check-sso` option. With this feature enabled, your browser will not perform a full redirect to the TideCloak server and back to your application, but this action will be performed in a hidden iframe. Therefore, your application resources are only loaded and parsed once by the browser, namely when the application is initialized and not again after the redirect back from TideCloak to your application. This approach is particularly useful in case of SPAs (Single Page Applications).

To enable the _silent_ `check-sso`, you provide a `silentCheckSsoRedirectUri` attribute in the init method. Make sure this URI is a valid endpoint in the application; it must be configured as a valid redirect for the client in the TideCloak Admin Console:

```javascript
await tidecloak.init({
    onLoad: 'check-sso',
    silentCheckSsoRedirectUri: `${location.origin}/silent-check-sso.html`
});
```

The page at the silent check-sso redirect uri is loaded in the iframe after successfully checking your authentication state and retrieving the tokens from the TideCloak server.
It has no other task than sending the received tokens to the main application and should only look like this:

```html
<!doctype html>
<html>
<body>
    <script>
        parent.postMessage(location.href, location.origin);
    </script>
</body>
</html>
```

Remember that this page must be served by your application at the specified location in `silentCheckSsoRedirectUri` and is _not_ part of the adapter.

> [!WARNING]
> _Silent_ `check-sso` functionality is limited in some modern browsers. Please see the [Modern Browsers with Tracking Protection Section](Modern-Browsers-with-Tracking-Protection).

To enable `login-required` set `onLoad` to `login-required` and pass to the init method:

```javascript
await tidecloak.init({
    onLoad: 'login-required'
});
```

After the user is authenticated the application can make requests to RESTful services secured by TideCloak by including the bearer token in the
`Authorization` header. For example:

```javascript
async function fetchUsers() {
    const response = await fetch('/api/users', {
        headers: {
            accept: 'application/json',
            authorization: `Bearer ${tidecloak.token}`
        }
    });

    return response.json();
}
```

One thing to keep in mind is that the access token by default has a short life expiration so you may need to refresh the access token prior to sending the request. You refresh this token by calling the `updateToken()` method. This method returns a Promise, which makes it easy to invoke the service only if the token was successfully refreshed and displays an error to the user if it was not refreshed. For example:

```javascript
try {
    await tidecloak.updateToken(30);
} catch (error) {
    console.error('Failed to refresh token:', error);
}

const users = await fetchUsers();
```

> [!NOTE]
> Both access and refresh token are stored in memory and are not persisted in any kind of storage. Therefore, these tokens should never be persisted to prevent hijacking attacks.

## Session Status iframe

By default, the adapter creates a hidden iframe that is used to detect if a Single-Sign Out has occurred. This iframe does not require any network traffic. Instead the status is retrieved by looking at a special status cookie. This feature can be disabled by setting `checkLoginIframe: false` in the options passed to the `init()` method.

You should not rely on looking at this cookie directly. Its format can change and it's also associated with the URL of the TideCloak server, not
your application.

> [!WARNING]
> Session Status iframe functionality is limited in some modern browsers. Please see [Modern Browsers with Tracking Protection Section](Modern-Browsers-with-Tracking-Protection).

## Implicit and hybrid flow

By default, the adapter uses the [Authorization Code](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) flow.

With this flow, the TideCloak server returns an authorization code, not an authentication token, to the application. The JavaScript adapter exchanges the `code` for an access token and a refresh token after the browser is redirected back to the application.

TideCloak also supports the [Implicit](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth) flow where an access token is sent immediately after successful authentication with TideCloak. This flow may have better performance than the standard flow because no additional request exists to exchange the code for tokens, but it has implications when the access token expires.

However, sending the access token in the URL fragment can be a security vulnerability. For example the token could be leaked through web server logs and or browser history.

To enable implicit flow, you enable the *Implicit Flow Enabled* flag for the client in the TideCloak Admin Console. You also pass the parameter `flow` with the value `implicit` to `init` method:

```javascript
await tidecloak.init({
    flow: 'implicit'
})
```

Note that only an access token is provided and no refresh token exists. This situation means that once the access token has expired, the application has to redirect to TideCloak again to obtain a new access token.

TideCloak also supports the [Hybrid](https://openid.net/specs/openid-connect-core-1_0.html#HybridFlowAuth) flow.

This flow requires the client to have both the *Standard Flow*  and *Implicit Flow*  enabled in the Admin Console. The TideCloak server then sends both the code and tokens to your application. The access token can be used immediately while the code can be exchanged for access and refresh tokens. Similar to the implicit flow, the hybrid flow is good for performance because the access token is available immediately.
But, the token is still sent in the URL, and the security vulnerability mentioned earlier may still apply.

One advantage in the Hybrid flow is that the refresh token is made available to the application.

For the Hybrid flow, you need to pass the parameter `flow` with value `hybrid` to the `init` method:

```javascript
await tidecloak.init({
    flow: 'hybrid'
});
```

## Hybrid Apps with Cordova

TideCloak supports hybrid mobile apps developed with [Apache Cordova](https://cordova.apache.org/). The adapter has two modes for this: `cordova` and `cordova-native`:

The default is `cordova`, which the adapter automatically selects if no adapter type has been explicitly configured and `window.cordova` is present. When logging in, it opens an [InApp Browser](https://cordova.apache.org/docs/en/latest/reference/cordova-plugin-inappbrowser/) that lets the user interact with TideCloak and afterwards returns to the app by redirecting to `http://localhost`. Because of this behavior, you whitelist this URL as a valid redirect-uri in the client configuration section of the Admin Console.

While this mode is easy to set up, it also has some disadvantages:

* The InApp-Browser is a browser embedded in the app and is not the phone's default browser. Therefore it will have different settings and stored credentials will not be available.
* The InApp-Browser might also be slower, especially when rendering more complex themes.
* There are security concerns to consider, before using this mode, such as that it is possible for the app to gain access to the credentials of the user, as it has full control of the browser rendering the login page, so do not allow its use in apps you do not trust.

The alternative mode  is`cordova-native`, which takes a different approach. It opens the login page using the system's browser. After the user has authenticated, the browser redirects back into the application using a special URL. From there, the TideCloak adapter can finish the login by reading the code or token from the URL.

You can activate the native mode by passing the adapter type `cordova-native` to the `init()` method:

```javascript
await tidecloak.init({
    adapter: 'cordova-native'
});
```

This adapter requires two additional plugins:

* [cordova-plugin-browsertab](https://github.com/google/cordova-plugin-browsertab): allows the app to open webpages in the system's browser
* [cordova-plugin-deeplinks](https://github.com/e-imaxina/cordova-plugin-deeplinks): allow the browser to redirect back to your app by special URLs

The technical details for linking to an app differ on each platform and special setup is needed.
Please refer to the Android and iOS sections of the [deeplinks plugin documentation](https://github.com/e-imaxina/cordova-plugin-deeplinks/blob/master/README.md) for further instructions.

Different kinds of links exist for opening apps:

* custom schemes, such as `myapp://login` or `android-app://com.example.myapp/https/example.com/login`.
* [Universal Links (iOS)](https://developer.apple.com/documentation/xcode/allowing-apps-and-websites-to-link-to-your-content/) / [Deep Links (Android)](https://developer.android.com/training/app-links/deep-linking).

While the former are easier to set up and tend to work more reliably, the latter offer extra security because they are unique and only the owner of a domain can register them. Custom-URLs are deprecated on iOS.  For best reliability, we recommend that you use universal links combined with a fallback site that uses a custom-url link.

Furthermore, we recommend the following steps to improve compatibility with the adapter:

* Universal Links on iOS seem to work more reliably with `response-mode` set to `query`
* To prevent Android from opening a new instance of your app on redirect add the following snippet to `config.xml`:

```xml
<preference name="AndroidLaunchMode" value="singleTask" />
```

## Custom Adapters

In some situations, you may need to run the adapter in environments that are not supported by default, such as Capacitor. To use the JavasScript client in these environments, you can pass a custom adapter. For example, a third-party library could provide such an adapter to make it possible to reliably run the adapter:

```javascript
import TideCloak from 'tidecloak-js';
import KeycloakCapacitorAdapter from 'keycloak-capacitor-adapter';

const tidecloak = new TideCloak({
    url: "http://tidecloak-server",
    realm: "my-realm",
    clientId: "my-app"
});

await tidecloak.init({
    adapter: KeycloakCapacitorAdapter,
});
```

This specific package does not exist, but it gives a pretty good example of how such an adapter could be passed into the client.

It's also possible to make your own adapter, to do so you will have to implement the methods described in the `TideCloakAdapter` interface. For example the following TypeScript code ensures that all the methods are properly implemented:

```javascript
import TideCloak, { TideCloakAdapter } from 'tidecloak-js';

// Implement the 'TideCloakAdapter' interface so that all required methods are guaranteed to be present.
const MyCustomAdapter: TideCloakAdapter = {
    async login(options) {
        // Write your own implementation here.
    }

    // The other methods go here...
};

const tidecloak = new TideCloak({
    url: "http://tidecloak-server",
    realm: "my-realm",
    clientId: "my-app"
});

await tidecloak.init({
    adapter: MyCustomAdapter,
});
```

Naturally you can also do this without TypeScript by omitting the type information, but ensuring implementing the interface properly will then be left entirely up to you.

## Modern Browsers with Tracking Protection

In the latest versions of some browsers, various cookies policies are applied to prevent tracking of the users by third parties, such as SameSite in Chrome or completely blocked third-party cookies. Those policies are likely to become more restrictive and adopted by other browsers over time. Eventually cookies in third-party contexts may become completely unsupported and blocked by the browsers. As a result, the affected adapter features might ultimately be deprecated.

The adapter relies on third-party cookies for Session Status iframe, _silent_ `check-sso` and partially also for regular (non-silent) `check-sso`. Those features have limited functionality or are completely disabled based on how restrictive the browser is regarding cookies. The adapter tries to detect this setting and reacts accordingly.

### Browsers with "SameSite=Lax by Default" Policy

All features are supported if SSL / TLS connection is configured on the TideCloak side as well as on the application side.  For example, Chrome is affected starting with version 84.

### Browsers with Blocked Third-Party Cookies

Session Status iframe is not supported and is automatically disabled if such browser behavior is detected by the adapter. This means the adapter cannot use a session cookie for Single Sign-Out detection and must rely purely on tokens. As a result, when a user logs out in another window, the application using the adapter will not be logged out until the application tries to refresh the Access Token. Therefore, consider setting the Access Token Lifespan to a relatively short time, so that the logout is detected as soon as possible. For more details, see: Session and Token Timeouts administration guide.

_Silent_ `check-sso` is not supported and falls back to regular (non-silent) `check-sso` by default. This behavior can be changed by setting `silentCheckSsoFallback: false` in the options passed to the `init` method. In this case, `check-sso` will be completely disabled if restrictive browser behavior is detected.

Regular `check-sso` is affected as well. Since Session Status iframe is unsupported, an additional redirect to TideCloak has to be made when the adapter is initialized to check the user's login status. This check is different from the standard behavior when the iframe is used to tell whether the user is logged in, and the redirect is performed only when the user is logged out.

An affected browser is for example Safari starting with version 13.1.

## API Reference

### Constructor

```javascript
// Recommended way to initialize the adapter.
import tcData from "/tidecloak.json";

new TideCloak({
  url: tcData['auth-server-url'],
  realm: tcData['realm'],
  clientId: tcData['resource'],
  vendorId: tcData['vendorId'],
  homeOrkUrl: tcData['homeOrkUrl']
});

// Alternatively a string to the path of the `tidecloak.json` file.
// Has some performance implications, as it will load the tidecloak.json file from the server.
new TideCloak("http://some-server/tidecloak.json");
// This way requies absolute trust in `some-server` and is therefore not recommended.
```

### Properties

**authenticated**
    Is `true` if the user is authenticated, `false` otherwise.

**token**
    The base64 encoded token that can be sent in the `Authorization` header in requests to services.

**tokenParsed**
    The parsed token as a JavaScript object.

**subject**
    The user id.

**idToken**
    The base64 encoded ID token.

**idTokenParsed**
    The parsed id token as a JavaScript object.

**realmAccess**
    The realm roles associated with the token.

**resourceAccess**
    The resource roles associated with the token.

**refreshToken**
    The base64 encoded refresh token that can be used to retrieve a new token.

**refreshTokenParsed**
    The parsed refresh token as a JavaScript object.

**timeSkew**
    The estimated time difference between the browser time and the TideCloak server in seconds. This value is just an estimation, but is accurate enough when determining if a token is expired or not.

**responseMode**
    Response mode passed in init (default value is fragment).

**flow**
    Flow passed in init.

**adapter**
    Allows you to override the way that redirects and other browser-related functions will be handled by the library.
    Available options:
    * "_default_" - the library uses the browser api for redirects (this is the default)
    * "_cordova_" - the library will try to use the InAppBrowser cordova plugin to load tidecloak login/registration pages (this is used automatically when the library is working in a cordova ecosystem)
    * "_cordova-native_" - the library tries to open the login and registration page using the phone's system browser using the BrowserTabs cordova plugin. This requires extra setup for redirecting back to the app (see (Hybrid Apps with Cordova)[Hybrid-Apps-with-Cordova] ).
    * "_custom_" - allows you to implement a custom adapter (only for advanced use cases)

**responseType**
    Response type sent to TideCloak with login requests. This is determined based on the flow value used during initialization, but can be overridden by setting this value.

## Methods

### init(options)

Called to initialize the adapter.

Options is an Object, where:

* **useNonce** - Adds a cryptographic nonce to verify that the authentication response matches the request (default is `true`).
* **onLoad** - Specifies an action to do on load. Supported values are `login-required` or `check-sso`.
* **silentCheckSsoRedirectUri** - Set the redirect uri for silent authentication check if onLoad is set to 'check-sso'.
* **silentCheckSsoFallback** - Enables fall back to regular `check-sso` when _silent_ `check-sso` is not supported by the browser (default is `true`).
* **token** - Set an initial value for the token.
* **refreshToken** - Set an initial value for the refresh token.
* **idToken** - Set an initial value for the id token (only together with token or refreshToken).
* **scope** - Set the default scope parameter to the TideCloak login endpoint. Use a space-delimited list of scopes. Those typically reference **Client scopes** defined on a particular client. Note that the scope `openid` will always be added to the list of scopes by the adapter. For example, if you enter the scope options `address phone`, then the request to TideCloak will contain the scope parameter `scope=openid address phone`. Note that the default scope specified here is overwritten if the `login()` options specify scope explicitly.
* **timeSkew** - Set an initial value for skew between local time and TideCloak server in seconds (only together with token or refreshToken).
* **checkLoginIframe** - Set to enable/disable monitoring login state (default is `true`).
* **checkLoginIframeInterval** - Set the interval to check login state (default is 5 seconds).
* **responseMode** - Set the OpenID Connect response mode send to TideCloak server at login request. Valid values are `query` or `fragment`. Default value is `fragment`, which means that after successful authentication will TideCloak redirect to JavaScript application with OpenID Connect parameters added in URL fragment. This is generally safer and recommended over `query`.
* **flow** - Set the OpenID Connect flow. Valid values are `standard`, `implicit` or `hybrid`.
* **enableLogging** - Enables logging messages from TideCloak to the console (default is `false`).
* **pkceMethod** - The method for Proof Key Code Exchange ([PKCE](https://datatracker.ietf.org/doc/html/rfc7636)) to use. Configuring this value enables the PKCE mechanism. Available options:
    - "_S256_" - The SHA256 based PKCE method (default)
    - _false_ - PKCE is disabled.
* **acrValues** - Generates the `acr_values` parameter which refers to authentication context class reference and allows clients to declare the required assurance level requirements, e.g. authentication mechanisms. See [Section 4. acr_values request values and level of assurance in OpenID Connect MODRNA Authentication Profile 1.0](https://openid.net/specs/openid-connect-modrna-authentication-1_0.html#acr_values).
* **messageReceiveTimeout** - Set a timeout in milliseconds for waiting for message responses from the TideCloak server. This is used, for example, when waiting for a message during 3rd party cookies check. The default value is 10000.
* **locale** - When onLoad is 'login-required', sets the 'ui_locales' query param in compliance with [section 3.1.2.1 of the OIDC 1.0 specification](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest).

Returns a promise that resolves when initialization completes.

### login(options)

Redirects to login form, returns a Promise.

Options is an optional Object, where:

* **redirectUri** - Specifies the uri to redirect to after login.
* **prompt** - This parameter allows to slightly customize the login flow on the TideCloak server side. For example, enforce displaying the login screen in case of value `login`. Or enforce displaying of consent screen for the value `consent` in case that client has `Consent Required`. Finally it is possible use the value `none` to make sure that login screen is not displayed to the user, which is useful just to check SSO for the case when user was already authenticated before (This is related to the `onLoad` check with value `check-sso` described above).
* **maxAge** - Used just if user is already authenticated. Specifies maximum time since the authentication of user happened. If user is already authenticated for longer time than `maxAge`, the SSO is ignored and he will need to re-authenticate again.
* **loginHint** - Used to pre-fill the username/email field on the login form.
* **scope** - Override the scope configured in `init` with a different value for this specific login.
* **idpHint** - Used to tell TideCloak to skip showing the login page and automatically redirect to the specified identity provider instead. More info in **Identity Provider documentation**.
* **acr** - Contains the information about `acr` claim, which will be sent inside `claims` parameter to the TideCloak server. Typical usage is for step-up authentication. Example of use `{ values: ["silver", "gold"], essential: true }`. See OpenID Connect specification
and **Step-up authentication documentation** for more details.
* **acrValues** - Generates the `acr_values` parameter which refers to authentication context class reference and allows clients to declare the required assurance level requirements, e.g. authentication mechanisms. See [Section 4. acr_values request values and level of assurance in OpenID Connect MODRNA Authentication Profile 1.0](https://openid.net/specs/openid-connect-modrna-authentication-1_0.html#acr_values).
* **action** - If the value is `register`, the user is redirected to the registration page. See **Registration requested by client section** for more details. If the value is `UPDATE_PASSWORD` or another supported required action, the user will be redirected to the reset password page or the other required action page. However, if the user is not authenticated, the user will be sent to the login page and redirected after authentication. See **Application Initiated Action section** for more details.
* **locale** - Sets the 'ui_locales' query param in compliance with **section 3.1.2.1 of the OIDC 1.0 specification**.
* **cordovaOptions** - Specifies the arguments that are passed to the Cordova in-app-browser (if applicable). Options `hidden` and `location` are not affected by these arguments. All available options are defined at https://cordova.apache.org/docs/en/latest/reference/cordova-plugin-inappbrowser/. Example of use: `{ zoom: "no", hardwareback: "yes" }`;

### createLoginUrl(options)

Returns a Promise containing the URL to login form.

Options is an optional Object, which supports same options as the function `login` .

### logout(options)

Redirects to logout.

Options is an Object, where:

* **redirectUri** - Specifies the uri to redirect to after logout.

### createLogoutUrl(options)

Returns the URL to log out the user.

Options is an Object, where:

* **redirectUri** - Specifies the uri to redirect to after logout.

### register(options)

Redirects to registration form. Shortcut for login with option action = 'register'

Options are same as for the login method but 'action' is set to 'register'

### createRegisterUrl(options)

Returns a Promise containing the url to registration page. Shortcut for createLoginUrl with option action = 'register'

Options are same as for the createLoginUrl method but `action` is set to `register`

### accountManagement()

Redirects to the Account Console.

### createAccountUrl(options)

Returns the URL to the Account Console.

Options is an Object, where:

* **redirectUri** - Specifies the uri to redirect to when redirecting back to the application.

### hasRealmRole(role)

Returns true if the token has the given realm role.

### hasResourceRole(role, resource)

Returns true if the token has the given role for the resource (resource is optional, if not specified clientId is used).

### loadUserProfile()

Loads the users profile.

Returns a promise that resolves with the profile.

For example:

```javascript
try {
    const profile = await tidecloak.loadUserProfile();
    console.log('Retrieved user profile:', profile);
} catch (error) {
    console.error('Failed to load user profile:', error);
}
```

### isTokenExpired(minValidity)

Returns true if the token has less than minValidity seconds left before it expires (minValidity is optional, if not specified 0 is used).

### updateToken(minValidity)

If the token expires within minValidity seconds (minValidity is optional, if not specified 5 is used) the token is refreshed. If -1 is passed as the minValidity, the token will be forcibly refreshed. If the session status iframe is enabled, the session status is also checked.

Returns a promise that resolves with a boolean indicating whether or not the token has been refreshed.

For example:

```javascript
try {
    const refreshed = await tidecloak.updateToken(5);
    console.log(refreshed ? 'Token was refreshed' : 'Token is still valid');
} catch (error) {
    console.error('Failed to refresh the token:', error);
}
```

### clearToken()

Clear authentication state, including tokens.
This can be useful if application has detected the session was expired, for example if updating token fails.

Invoking this results in onAuthLogout callback listener being invoked.

## Callback Events

The adapter supports setting callback listeners for certain events. Keep in mind that these have to be set before the call to the `init()` method.

For example:
```javascript
tidecloak.onAuthSuccess = () => console.log('Authenticated!');
```

The available events are:

* *onReady(authenticated)* - Called when the adapter is initialized.
* *onAuthSuccess* - Called when a user is successfully authenticated.
* *onAuthError* - Called if there was an error during authentication.
* *onAuthRefreshSuccess* - Called when the token is refreshed.
* *onAuthRefreshError* - Called if there was an error while trying to refresh the token.
* *onAuthLogout* - Called if the user is logged out (will only be called if the session status iframe is enabled, or in Cordova mode).
* *onTokenExpired* - Called when the access token is expired. If a refresh token is available the token can be refreshed with updateToken, or in cases where it is not (that is, with implicit flow) you can redirect to the login screen to obtain a new access token.
