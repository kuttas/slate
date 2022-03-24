---
title: Plex API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - http
  - shell
  - go

toc_footers:
  - <a href='https://www.userclouds.com'>Copyright UserClouds 2021-</a>

search: true

code_clipboard: true

meta:
  - name: description
    content: Documentation for the Plex API
---

# Introduction

Welcome to the Plex API, an OIDC/OAuth-conformant authentication API with support for multiple underlying IDentity Providers (IDPs) such as:

- UserClouds
- Auth0
- Cognito (coming soon)

We have documentation & samples for:

- Pure HTTP
- Shell (via curl)
- Golang (native SDK)
- Python (coming soon)
- Typescript (coming soon)

You can view code examples in the dark area to the right, and you can switch the programming language of the examples with the tabs in the top right.

# Authentication

## Login via Authorization Code (with or without Proof Key Code Exchange)

> To start the OIDC Authorization Code Login flow, use this code:

```http
GET http://sample.tenant.userclouds.com/oidc/authorize?
  response_type=code&
  client_id=<YOUR_CLIENT_ID>&
  redirect_uri=<YOUR_CALLBACK_URL>&
  scope=openid+<ADDITIONAL_SCOPES>&
  state=<STATE>&
  code_challenge_method=S256&
  code_challenge=<CODE_CHALLENGE>
```

```shell
curl 'https://sample.tenant.userclouds.com/oidc/authorize?response_type=code&client_id=<YOUR_CLIENT_ID>&redirect_uri=<YOUR_CALLBACK_URL>&scope=openid+<ADDITIONAL_SCOPES>&state=<STATE>&code_challenge=<CODE_CHALLENGE>&code_challenge_method=S256'
```

```go
import "golang.org/x/oauth2"

prov, _ := oidc.NewProvider(context.Background(), "https://<YOURTENANTNAME>.tenant.userclouds.com")

oauthConfig := oauth2.Config{
  ClientID:     "<YOUR_CLIENT_ID>",
  Endpoint:     prov.Endpoint(),
  RedirectURL:  "<YOUR_CALLBACK_URL>",
  Scopes:       []string{"openid <ADDITIONAL_SCOPES>"},
}

// Authorization Code Flow without Proof Key Code Exchange (PKCE)
url := oauthConfig.AuthCodeURL("<STATE>")

// -- OR --

// Authorization Code Flow *with* Proof Key Code Exchange (PKCE) using SHA256 (the only supported method)
url := oauthConfig.AuthCodeURL(
  state,
  oauth.SetAuthURLParam("code_challenge_method", "S256"),
  oauth.SetAuthURLParam("code_challenge", "<CODE_CHALLENGE>"))

// Redirect user agent to URL authorize endpoint
http.Redirect(w, r, url, http.StatusTemporaryRedirect)
```

> Make sure to replace the values in brackets (e.g. Tenant URL, Client ID, State, Additional Scopes, and Redirect URL) with values appropriate to your application.
> Plex tenants, clients, and redirect URLs are registered on the UserClouds [developer console](https://console.prod.userclouds.com).

The Authorization Code flow is the most commonly used and recommended way for client applications (web, mobile) to allow users to log in.

If the application can be trusted to hold a secret (e.g. a web application with secure backend), the standard Authorization Code flow can be used.

If the application cannot securely hold a secret (e.g. web single-page app or mobile app), and particularly if access tokens are needed, it should utilize the Proof Key Code Exchange feature to prevent token hijacking.

<aside class="notice">
If only ID tokens are needed, an alternate login flow (Implicit Flow with Form post) may also be used.
</aside>

### Request

Because this is a `GET` request, all parameters should be specified as query parameters in the URL.

| Parameter             | Type   | Required? | Description                                                                                                                                                                                                                                                                                     |
| --------------------- | ------ | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| response_type         | string | yes       | `code`                                                                                                                                                                                                                                                                                          |
| client_id             | string | yes       | The Client ID of a registered app/client in your UserClouds tenant.                                                                                                                                                                                                                             |
| redirect_uri          | string | yes       | A URL to your application. Must use `https://` for production web apps, or a mobile custom scheme for mobile apps. Can use `http://` for localhost and development tenants.                                                                                                                     |
| scope                 | string | yes       | Must specify `openid` in order to get an ID token from the token exchange endpoint later. May also specify `profile`, `email`, or other common scopes (see <a href='https://openid.net/specs/openid-connect-core-1_0.html#ScopeClaims'>OIDC Spec</a> for details).                              |
| state                 | string | yes       | An application-defined value used for 2 purpose: encode application state/context for a request, and to prevent CSRF attacks. See <a href='https://tools.ietf.org/id/draft-ietf-oauth-security-topics-13.html#rec_redirect'>OAuth Security Topics</a> for details.                              |
| code_challenge_method | string | no        | Indicates that PKCE should be used with a given method. Only 'S256' (SHA 256) is currently supported.                                                                                                                                                                                           |
| code_challenge        | string | no        | A value generated by applying `code_challenge_method` (i.e. SHA 256) to a `code_verifier` value generated by the client. See the <a href='https://datatracker.ietf.org/doc/html/rfc7636'>PKCE spec</a> for more details. NOTE: this value is required iff `code_challenge_method` is specified. |

### Response

```
# On Success

HTTP/1.1 302 Found

Location: https://<YOUR_CALLBACK_URL>?code=<AUTHORIZATION_CODE>&state=<STATE>

# On Error

HTTP/1.1 302 Found

Location: https://<YOUR_CALLBACK_URL>?error=<OAUTH_ERROR_TYPE>&error_description=<HUMAN_READABLE_DESCRIPTION>
```

If the request is valid, the immediate `GET` call to the authorize endpoint will redirect to Plex's internal endpoint(s) for the user to perform an interactive login. After the user sucessfully authenticates (or in case of an error in the process), the user agent will ultimately be redirected (HTTP 302) to the application-provided (and pre-registered) redirect URL, unless the redirect URL and/or Client ID are invalid (in which case the response will be a 400 Bad Request).

| Parameter         | Type   | Condition | Description                                                                                                                                                           |
| ----------------- | ------ | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| code              | string | success   | Cryptographically secure token which can be exchanged with the Token endpoint.                                                                                        |
| state             | string | success   | The application-defined value passed in to the Authorize call. The client should validate that this matches to prevent attacks (hijacks, replays, etc.).              |
| error             | string | failure   | One of the OAuth/OIDC-spec defined error values. See the <a href='https://datatracker.ietf.org/doc/html/rfc6749#section-4.1.2.1'>OAuth 2.0 spec</a> for more details. |
| error_description | string | failure   | A human readable explanation of the error.                                                                                                                            |

## Token Exchange for Authorization Code Flow (with or without Proof Key Code Exchange)

> To exchange an authorization code for an access and/or ID token, use this code:

```http
POST http://sample.tenant.userclouds.com/oidc/token
Content-Type:
  grant_type=authorization_code&
  code=<CODE>&
  redirect_uri=<YOUR_CALLBACK_URL>&
  code_verifier=<CODE_VERIFIER>
```

```shell
curl --request POST \
  --url 'https://sample.tenant.userclouds.com/oidc/token' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data '{"response_type":"code", "client_id":"<YOUR_CLIENT_ID>", "redirect_uri":"<YOUR_CALLBACK_URL>", "scope":"openid <ADDITIONAL_SCOPES>", "state":"<STATE>", "code_challenge":"<CODE_CHALLENGE>", "code_challenge_method":"S256"}'

```

```go
import "golang.org/x/oauth2"

prov, _ := oidc.NewProvider(context.Background(), "https://<YOURTENANTNAME>.tenant.userclouds.com")

oauthConfig := oauth2.Config{
  ClientID:     "<YOUR_CLIENT_ID>",
  ClientSecret: "<YOUR_CLIENT_SECRET>",
  Endpoint:     prov.Endpoint(),
  RedirectURL:  "<YOUR_CALLBACK_URL>",
}

// Authorization Code Flow without Proof Key Code Exchange (PKCE)
tokenResponse, err := oauthConfig.Exchange(ctx, "<CODE>")

// -- OR --

// Authorization Code Flow *with* Proof Key Code Exchange (PKCE) using SHA256 (the only supported method)
tokenResponse, err := oauthConfig.Exchange(ctx, "<CODE>", oauth.SetAuthURLParam("code_verifier", "<CODE_VERIFIER>"))

if err { ... }

rawIDToken, ok := tokenResponse.Extra("id_token").(string)
if !ok { ... }

accessToken := tokenResponse.AccessToken
```

### Request

This is a `POST` request with all parameters form encoded in the body.

| Parameter     | Type   | Required? | Description                                                                                                                                                                                                           |
| ------------- | ------ | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| grant_type    | string | yes       | `authorization_code`                                                                                                                                                                                                  |
| client_id     | string | yes       | The Client ID of a registered app/client in your UserClouds tenant.                                                                                                                                                   |
| client_secret | string | yes       | The Client Secret of a registered app/client in your UserClouds tenant.                                                                                                                                               |
| redirect_uri  | string | yes       | A URL to your application. Must use `https://` for production web apps, or a mobile custom scheme for mobile apps. Can use `http://` for localhost and development tenants.                                           |
| code_verifier | string | no        | If PKCE was used to issue the authorization code, then this value must be specified and valid for the token exchange. See the <a href='https://datatracker.ietf.org/doc/html/rfc7636'>PKCE spec</a> for more details. |

### Response

```
# TODO: refresh token support, token expiry

HTTP/1.1 200 OK
Content-Type: application/json
{
  "access_token":"<ACCESS_TOKEN>",
  "id_token":"<ID_TOKEN>",
  "token_type":"Bearer",
}
```

If the request is valid, the endpoint returns a json struct with an access token and/or ID token (depending on the scopes of the original request), as well as the token type which is always "Bearer".

| Parameter    | Type   | Condition | Description                                                                      |
| ------------ | ------ | --------- | -------------------------------------------------------------------------------- |
| access_token | string | success   | Cryptographically secure token which can be used to make API calls.              |
| id_token     | string | success   | A raw three-part JWT which contains information about the resource owner / user. |
| token_type   | string | success   | Always "Bearer"                                                                  |

# Invites

## Send Invite
