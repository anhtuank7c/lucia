---
title: "Built-in providers"
description: "Learn how to use the built-in OAuth providers"
---

This page will use Github OAuth but the API and auth flow is nearly identical between providers. See the each provider's documentation (from the sidebar) to see all the available providers.

## Setup

Initialize the handler using the Lucia `Auth` instance and provider-specific config.

```ts
// see provider docs for imports
import { github } from "@lucia-auth/oauth/providers";

export const auth = lucia({
	// ...
});

export const githubAuth = github(auth, config);
```

## Get authorization url

You can create a new authorization url with `getAuthorizationUrl()`. It will return the url as the first item and, in general, an OAuth state as the second. However, depending on the provider, the state may not be returned, or it may return a PKCE code verifier in addition to the state. Please check each provider's docs.

The state should be stored as a http-only cookie.

```ts
import { auth, githubAuth } from "$lib/lucia.js";

// get url to redirect the user to, with the state
const [url, state] = await githubAuth.getAuthorizationUrl();

setCookie("github_oauth_state", state, {
	path: "/",
	httpOnly: true, // only readable in the server
	maxAge: 60 * 60 // a reasonable expiration date
});

// redirect to authorization url
redirect(url);
```

## Handle callback

Upon authentication, the provider will redirect the user back to your application. The url usually includes a state, which should be the same as the cookie we set in the previous step.

### Validate callback

Validate the code using `validateCallback()`. If the code is valid, this will return a new [`ProviderUserAuth`](/reference/oauth/interfaces#provideruserauth)among provider specific items (such as provider user data and access tokens).

```ts
import { auth, githubAuth } from "$lib/lucia.js";

const code = requestUrl.searchParams.get("code");
const state = requestUrl.searchParams.get("state");

// get state cookie we set when we got the authorization url
const stateCookie = getCookie("github_oauth_state");

// validate state
if (!state || !storedState || state !== storedState) throw new Error(); // invalid state

try {
	await githubAuth.validateCallback(code);
} catch {
	// invalid code
}
```

> (red) **There's no guarantee that the user's email provided by the provider is verified!** Make sure to check with each provider's documentation if you're working with email. This is super important if you're planning to link accounts.

### Authenticate user with Lucia

`validateCallback()` returns a few properties and method. [`ProviderUserAuth.existingUser`](/reference/oauth/interfaces#provideruserauth) will be defined if a Lucia user already exists for the authenticated provider account. This is based on the provider user id (e.g. Github user id) and not shared identifiers like email.

If not, you can create a new Lucia user linked to the provider with [`ProviderUserAuth.createUser()`](/reference/oauth/interfaces#createuser). You can get the provider user data with `githubUser` for Github, etc.

```ts
const { existingUser, createUser, githubUser } =
	await githubAuth.validateCallback(code);
const getUser = async () => {
	if (existingUser) return existingUser;
	// create a new user if the user does not exist
	return await createUser({
		attributes: {
			username: githubUser.login
		}
	});
};
const user = await getUser();

// login user
const session = await auth.createSession({
	userId: user.userId,
	attributes: {}
});
const authRequest = auth.handleRequest();
authRequest.setSession(session); // store session cookie
```

### Add provider to existing user

Alternatively, you may want to add a new authentication method to an existing user by creating a new key. Calling [`ProviderUserAuth.createKey()`](/reference/oauth/interfaces#createkey) will create a new key linked to the provided user id.

```ts
const { existingUser, createKey } = await githubAuth.validateCallback(code);

if (!existingUser) {
	await createKey(currentUser.userId);
}
```

See [OAuth account linking](/guidebook/oauth-account-linking) guide for details.

### Get API tokens

Most provider also includes access tokens and refresh tokens in the `validateCallback()`.

```ts
const { githubTokens } = await githubAuth.validateCallback(code);
githubTokens.accessToken;
githubTokens.accessTokenExpiresIn;
```

## Errors

Request errors are thrown as [`OAuthRequestError`](/reference/oauth/interfaces#oauthrequesterror), which includes a request and response object.

```ts
import { OAuthRequestError } from "@lucia-auth/oauth";

try {
	await githubAuth.validateCallback(code);
} catch (e) {
	if (e instanceof OAuthRequestError) {
		const { request, response } = e;
	}
}
```
