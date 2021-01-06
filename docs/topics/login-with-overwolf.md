---
id: login-with-overwolf
title: App login with Overwolf
sidebar_label: Login with Overwolf
---

This article will explain how to implement an Overwolf login/auth interface in your Website. 

## Login flow overview

This flow is web browser flow only, as currently, we do not offer client SDK that supports API-based authentication.

* Developers register their app for OW SSO and get unique client_id and client_secret.
* The actual login is done using a login form hosted on the OW servers.  This means you should implement on your Website only a "Login with Overwolf" button that opens a new window/tab with the OW login form. (More info in [step 1: Engage the SSO flow](#1-engage-the-sso-flow)).
* Once the login is completed on the OW hosted login page, the user is redirected to a pre-defined redirect_URI hosted on YOUR server (More info on how to implement this page in ["Create redirect_uri endpoint"](#create-redirect_uri-endpoint)).
* The redirect_URI page executes a POST request to OW servers to request and get the auth token that was created after the login (More info in [step 3: Get the auth token](#3-get-the-auth-token)).

## Prerequisite

### Register your app on Overwolf

To implement OW login in your Website, you first need to register your app on Overwolf (send us an email to developers@overwolf.com) to generate your `client_id` and `client_secret`.

You should provide these parameters:

1. client_name - The app's name.
2. redirect_uris - An endpoint hosted on your server. More details [here](#create-redirect_uri-endpoint).  
3. logo_uri - URL of the app's logo.
4. policy_uri - URL of the app's privacy policy.
5. tos_uri - URL of the app's "Terms of Service".

Once the registration is completed, you will get your app's `client_id` and `client_secret`.

### Create redirect_uri endpoint

On your server, create an endpoint used to get the auth-token from Overwolf (by making a POST request to the OW server).  

```js
POST https://accounts.overwolf.com/oauth2/token?client_id={client id}&client_secret={client secret}&grant_type=authorization_code&code={code that came from request object, e.g: request.query.code}&redirect_uri={redirect_uri}
```

#### Required Query params:

* client_id
* client_secret
* grant_type
* code 
* redirect_uri

#### Required HTTP headers:

* Content-Type: "application/x-www-form-urlencoded"

#### Sample code

This is an example code that shows how to implement a redirect_uri endpoint on your server.

```js
var express = require('express');
var router = express.Router();
const axios = require('axios');
const querystring = require('querystring');

const demoClient = {
  client_id: 'xxxxxxxx',
  client_secret: 'yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy',
};

/**
 * this is the callback endpoint as passed in the redirect_uri parameter
 * and should be whitelisted in the oauth client application
 */
router.get('/oidc-callback', function(req, res, next) {
  const client = demoClient;
  axios.post('https://accounts.overwolf.com/oauth2/token',
    querystring.stringify({
      client_id: client.client_id,
      client_secret: client.client_secret,
      grant_type: 'authorization_code',
      code: req.query.code,
      redirect_uri: 'http://localhost:5000/oidc-callback'
    }),
    {
      headers: {
        "Content-Type": "application/x-www-form-urlencoded"
      }
    }).then(function(response) {
      // the response will contain the access token to be used
      console.log(response);
      res.json({});
    }).catch((e) => {
      console.error(e)
      res.send('err')
  });
});

router.get('/oidcresult', function(req, res, next) {
  res.json(req.query);
});

module.exports = router;
```

As you can see, `redirect_uri` is `http://localhost:5000/oidc-callback`. This page receives the auth token (or login error) once the auth process is finished on the OW side.


## 1. Engage the SSO flow.

The first step is to implement a login button on your Website.  
The button click should open a new tab or popup window by implementing this GET request:

```js
GET https://accounts.overwolf.com/oauth2/auth?response_type=code&client_id={client id}&redirect_uri={redirect_uri}&scope={desired scope separated by '+', e.g: openid+profile+email}
```  

#### Required Query params:

* response_type
* client_id
* redirect_uri
* scope

## 2. Login on Overwolf

Once the SSO flow is engaged, the user is redirected to the OW hosted login page:

![OW login screenshot](assets/ow_login.png)

On successful login, the user also gets a consent screen to requested scopes:

![OW login consent](assets/ow_login_consent.png)

After the consent, we will redirect the user back to the redirect_uri pre-defined in the registration process.

## 3. Get the auth token

If there was no error in the flow, the user is redirected to the redirect_uri endpoint, that POST request for the auth-token.  
You can see how this POST request looks like in the [Create redirect_uri endpoint
](#create-redirect_uri-endpoint) section.

Once the POST completed, you will get the following auth token details:

* access_token
* expires_in
* id_token
* scope
* token_type

## 4. Close the login window.

Now, we can safely close the login window.

The login process is complete.