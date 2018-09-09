
# TL;DR

We show how to use OAuth 2.0 securely when using

- a [Classic Web Application](#classic-web-application-authorization-code-grant-flow),
- a [Single Page Application](#single-page-application-implicit-grant-flow), and
- a [Mobile Application](#mobile-application-authorization-code-grant-with-pkce)

as clients. For each of these clients, we  elaborate on the overall design, implement that design using the MEAN stack, and touch upon common security mistakes.

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [TL;DR](#tldr)
- [Introduction](#introduction)
- [Running Example and Background](#running-example-and-background)
- [Architect: Major Design Decisions](#architect-major-design-decisions)
  - [Use the Authorization Code Grant for Classic Web Applications and Native Mobile Apps](#use-the-authorization-code-grant-for-classic-web-applications-and-native-mobile-apps)
  - [Use Refresh Tokens When You Trust the Client to Store Them Securely](#use-refresh-tokens-when-you-trust-the-client-to-store-them-securely)
  - [Use Handle-Based Tokens Outside Your Network](#use-handle-based-tokens-outside-your-network)
  - [Selecting the Token Type](#selecting-the-token-type)
  - [Use Bearer Tokens When You do not Care to Whom They Were Issued](#use-bearer-tokens-when-you-do-not-care-to-whom-they-were-issued)
  - [Combining Authorization Server and Resource Server](#combining-authorization-server-and-resource-server)
- [Classic Web Application: Authorization Code Grant Flow](#classic-web-application-authorization-code-grant-flow)
  - [Design](#design)
  - [Insecure Implementation](#insecure-implementation)
    - [Gallery](#gallery)
      - [Step 1: Click print Button](#step-1-click-print-button)
      - [Step 2: Redirect to Gallery, Authenticate, and Consent Request](#step-2-redirect-to-gallery-authenticate-and-consent-request)
      - [Step 3: Approval and Generation of Authorization Code](#step-3-approval-and-generation-of-authorization-code)
      - [Step 4: Redirection To The Client](#step-4-redirection-to-the-client)
      - [Step 5:  Client Sends Authorization Code And Its Credentials To The Token Endpoint](#step-5-client-sends-authorization-code-and-its-credentials-to-the-token-endpoint)
      - [Step 6: Issue An Access Token At The Token Endpoint](#step-6-issue-an-access-token-at-the-token-endpoint)
      - [Step 7: Access a Resource](#step-7-access-a-resource)
    - [Print](#print)
  - [Security Considerations](#security-considerations)
    - [Gallery](#gallery)
      - [Token Endpoint: Bind the Authorization Code to the RedirectURI](#token-endpoint-bind-the-authorization-code-to-the-redirecturi)
- [Mobile Application: Authorization Code Grant with PKCE](#mobile-application-authorization-code-grant-with-pkce)
  - [Design](#design)
  - [Implementation](#implementation)
  - [Testing](#testing)
- [Single Page Application: Implicit Grant Flow](#single-page-application-implicit-grant-flow)
  - [Design](#design)
  - [Implementation](#implementation)
  - [Testing](#testing)
- [First Party Mobile Application: Resource Owner Password Credentials Flow](#first-party-mobile-application-resource-owner-password-credentials-flow)
  - [Design](#design)
  - [Implementation](#implementation)
  - [Security Considerations](#security-considerations)
    - [OpenID Connect](#openid-connect)
- [Checklists](#checklists)
  - [For Architects](#for-architects)
  - [For Software Engineers](#for-software-engineers)
  - [For Testers](#for-testers)
- [Conclusion](#conclusion)

<!-- /TOC -->

# Introduction

In this article, we elaborate on *common* security mistakes that architects and developers make when designing or implementing OAuth 2.0-enabled applications. The article not only describes these mistakes from a theoretical perspective, but also provides a set of working sample applications that contain those mistakes. This serves three purposes:

1. developers are able to identify a missing security control and learn how to implement it securely.
1. architects and developers are able to assess the impact of not implementing a security control.
1. Testers are able to identify the mistakes in a running application.

The article is structured as follows. Section [Background](#background) introduces the OAuth 2.0 Protocol using a running example. The subsequent sections show how to use OAuth 2.0 when using a [Classic Web Application](#classic-web-application-authorization-code-grant-flow), a [Single Page Application](#single-page-application-implicit-grant-flow), and [Mobile Application](#mobile-application-authorization-code-grant-with-pkce) as clients. For each of these sections, we  elaborate on the overall design, implement that design using the MEAN stack, and touch upon common security mistakes. Section [Checklists](#checklists) summarizes this article in the form of checklists for architects, developers, and testers. Finally, Section [Conclusion](#conclusion) concludes.

**Note:** the mistakes are common across technology stacks; we use the MEAN stack for illustration purposes only.

# Running Example and Background

Our canonical running example consists of a web site that enables users to manage pictures, named `gallery`.  This gallery application is similar to `flickr.com` in the sense that users can upload pictures, share them with friends, and organize those pictures in different albums.

 As our gallery application became quite popular, we got requests from various companies to integrate with our `gallery` application. To that end, we decided to open up the `REST API` that forms the foundation of our application towards those companies. These companies use the following types of clients:
- a third-party website that allows users to print the pictures hosted at our gallery site, named `photoprint`.
- a third-party mobile application that enables users to view pictures from many gallery applications.
- a website of a printing company that has been designed as a single page application.

Naturally, we also would like to create our own mobile application that our users can use to access our gallery site. However, as we are concerned about security, users should be able to give those third-party applications permission to access their pictures without providing their username and password to those applications. It seems that the OAuth 2.0 protocol might help achieve our goals.

![Our running example consists of a photo gallery API that can be accessed by many applications](./pics/RunningExample.png)

[OAuth 2.0](https://tools.ietf.org/html/rfc6749) is a [standard](https://tools.ietf.org/html/rfc6750) that enables users to give websites access to their data/services at other websites. For instance, a user gives a photo printing website access to her pictures on Flickr. Before performing a deep-dive into the specifics of OAuth 2.0, we introduce some definitions (taken from auth0):

- ***Resource Owner***: the entity that can grant access to a protected resource. Typically this is the end-user.
- ***Client***: an application requesting access to a protected resource on behalf of the Resource Owner. This is also called a Relying Party.
- ***Resource Server***: the server hosting the protected resources. This is the API you want to access, in our case `gallery`.
- ***Authorization Server***: the server that authenticates the Resource Owner, and issues access tokens after getting proper authorization. This is also called an identity provider (IdP).
- ***User Agent***: the agent used by the Resource Owner to interact with the Client, for example a browser or a  mobile application.

Before opening up our gallery API towards third-parties, we would need to make the following high-level design decisions.

# Architect: Major Design Decisions

Now that we know what OAuth 2.0 is typically used for, we elaborate on the major design decisions that architects face when designing an OAuth 2.0 enabled application.

## Use the Authorization Code Grant for Classic Web Applications and Native Mobile Apps

In OAuth 2.0, the interactions between the user and her browser, the Authorization Server, and the Resource Server can be performed in four different flows.

1. the ***authorization code grant***: the *Client* redirects the user (*Resource Owner*) to an *Authorization Server* to ask the user whether the *Client* can access her *Resources*. After the user confirms, the *Client* obtains an *Authorization Code* that the *Client* can exchange for an *Access Token*. This *Access Token* enables the *Client* to access the *Resources* of the *Resource Owner*.
1. the ***implicit grant*** is a simplification of the authorization code grant. The Client obtains the *Access Token* directly rather than being issued an *Authorization Code*.
1. the ***resource owner password credentials grant*** enables the *Client* to obtain an *Access Token* by using the username and password of the *Resource Owner*.
1. the ***client credentials grant*** enables the *Client* to obtain an Access Token by using its own credentials.

Do not worry if you do not understand the flows right away. They are elaborated upon in detail in subsequent sections. What you should remember is that:

- *Clients* can obtain *Access Tokens* via four different flows.
- *Clients* use these access tokens to access an API.

A major design decision is deciding which flows to support. This largely depends on the type of clients the application supports. [Auth0 provides an excellent flow chart that helps making a good  decision](https://auth0.com/docs/api-auth/which-oauth-flow-to-use). In summary, if the *Client* is:

- A classic web application, use the Authorization Code Grant.
- A single page application, use the Implicit Grant.
- A native mobile application, use the Authorization Code Grant with PKCE.
- A client that is absolutely trusted with user credentials (i.e. the Facebook app accessing Facebook), use the Resource Owner Password Grant.
- A client that is the owner of the data, use the Client Credentials Grant.

![A decision tree that helps an architect to decide which OAuth 2.0 flows to support](./pics/DecisionTreeFlow.png)

Using an incorrect flow for a client has various security implications. For instance, using the Resource Owner Password Grant for third-party mobile applications, gives those mobile applications access to the user's credentials. These credentials allow those applications to access all of the user data, as they can just login as the user itself. This is probably something you want to avoid.

In the subsequent sections, we show how to use OAuth 2.0 when using a [Classic Web Application](#classic-web-application-authorization-code-grant-flow), a [Single Page Application](#single-page-application-implicit-grant-flow), and [Mobile Application](#mobile-application-authorization-code-grant-with-pkce) as clients. For each of these sections, we  elaborate on the overall design, implement that design using the MEAN stack, and touch upon common security mistakes.

## Use Refresh Tokens When You Trust the Client to Store Them Securely

OAuth 2.0 uses two types of tokens, namely Access Tokens and Refresh Tokens.

- ***Access Tokens*** are tokens that the *Client* gives to the API to access *Resources*.
- ***Refresh Tokens*** are tokens that the *Client* can use to obtain a new *Access Token* once the old one expires.

Think of Access Tokens [like a session that is created once you authenticate to a website](https://nordicapis.com/api-security-oauth-openid-connect-depth/). As long as that session is valid, we can interact with that website without needing to login again. Once the session times out, we would need to login again with our username and password. Refresh tokens are like that password, as they allow a *Client* to create a new session.

Just like passwords, it is important that the *Client* stores these *Refresh Tokens* securely. If you do not trust that the client will store those tokens securely, do not issue *Refresh Tokens*. An attacker with access to the *Refresh Tokens* can obtain new *Access Tokens* and use those *Access tokens* to access a user's *Resources*. The main downside of not using *Refresh Tokens* is that users would need to re-authenticate every time the *Access Token* expires.

![A decision tree that helps an architect to decide whether to support refresh tokens](./pics/DecisionTreeRefreshToken.png)

## Use Handle-Based Tokens Outside Your Network

There are two ways to pass the above tokens throughout the system, namely:

- ***Handle-based Tokens*** are typically random strings that can be used to retrieve the data associated with that token. This is similar to passing a variable by reference in a programming language.
- ***Self-contained Tokens*** are tokens that contain all the data associated with that token. This is similar to passing a variable by value in a programming language.

To increase maintainability, [use self-contained tokens within your network, but use handle-based tokens outside of it](https://www.youtube.com/watch?v=BdKmZ7mPNns). The main reason is that *Access Tokens* are meant to be consumed by the API itself, not by the *Client*. If we share self-contained tokens with Clients, they might consume these tokens in ways that we had not intended. Changing the content (or structure) of our self-contained tokens might thus break lots of *Client* applications (in ways we had not foreseen).

If *Client applications* require access to data that is in the self-contained token, offer an API that enables *Client* applications to obtain information related to that *Access Token*. If we want to use self-contained tokens and prevent clients from accessing the contents, encrypt those tokens.

![A decision tree that helps an architect decide whether to support handle-based tokens](./pics/handle-based-tokens.png)

## Selecting the Token Type

The OAuth 2.0 spec does not specify the types of tokens that can be used and as such we, as architects, must select a token type that makes sense for our application. Applications typically use on of the following token types:

- ***[JSON Web Tokens (JWTs)](https://tools.ietf.org/html/rfc7523)*** are self-contained tokens expressed as a [JavaScript Object Notation (JSON)](https://tools.ietf.org/html/rfc7159) structure. JWTs are part of the [JSON Identity Suite](https://www.slideshare.net/2botech/the-jsonbased-identity-protocol-suite). Other things in this suite include [JSON Web Algorithms (JWA)](https://tools.ietf.org/html/rfc7518) for expressing algorithms, [JSON Web Key (JWK)](https://tools.ietf.org/html/rfc7517) for representing keys, [JSON Web Encryption (JWE)](https://tools.ietf.org/html/rfc7516) for encryption, and [JSON Web Signature (JWS)](https://tools.ietf.org/html/rfc7515) for signatures. These RFCs are typically used by OAuth applications as well as by [OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html).
- ***Custom tokens*** are typically handle-based tokens that store non-standardized information associated with the handle in a database.
- ***[WS-Security tokens](https://tools.ietf.org/html/rfc7522)*** and SAML tokens are typically supported for applications that rely extensively on SAML.

Selecting the token type depends on application constraints. For instance, if the application heavily relies on SAML tokens, it might be best to use the SAML profile. Otherwise, JWTs or custom tokens might be used. Usage of JWTs as  OAuth 2.0 Bearer tokens is recommended when the APIs of the application have been implemented by various partners; e.g. an application using a Micro Service Architecture where the different services are offered by different companies. Each of those token types will have different security considerations to use them securely, but those considerations are not a major factor in selecting one.

![A decision tree that helps an architect to select a token type](./pics/selecting-token-type.png)

## Use Bearer Tokens When You do not Care to Whom They Were Issued

OAuth supports different token profiles, namely:

- ***[Bearer tokens](https://tools.ietf.org/html/rfc6750)*** are like concert tickets: the venue does not care to who the ticket was issued; it lets you attend the concert if you have the ticket regardless of who bought it originally. The API (*Resource Server*) does not care who presents the Bearer token, it gives the *Client* access if it is a valid token.
- ***[Holder of Key tokens](https://tools.ietf.org/html/rfc7800)*** are like debit cards: you can only use it if you provide a PIN that unlocks the card. This PIN assures the merchant that the one using the card is the one to whom it was issued. Holder of Key Tokens cannot be used without proof of possession.

For most applications, Bearer tokens are sufficient. The main downside of using them is that if attackers obtains *Access Tokens*, they can use them.

## Combining Authorization Server and Resource Server

The OAuth 2.0 protocol defines the concepts of an *Authorization Server* (where a user gives consent to allow an application to access its resources) and a *Resource Server* (i.e. the API that an application can access). Both servers can be deployed on the same server or they also may be deployed to different servers.

If you are using a micro-service oriented architecture, it is best practice to separate both server types into different entities, otherwise they can be merged into one.

![A decision tree that helps an architect to decide whether to combine authorization and resource server](./pics/combining-authorization-resource-server.png)

# Classic Web Application: Authorization Code Grant Flow

In this Section, we elaborate on using OAuth 2.0 with a classic web application as Client: we introduce the overall design, implement that design, and touch upon common security mistakes made during design and implementation.

In our running example, the third-party website `photoprint` enables users to print the pictures hosted at our gallery site uses this flow. In the real world, we typically do not control how the `photoprint` application uses our API, and therefore we stipulate the (security) requirements that our partner `photoprint` must implement in a Business Requirements Document (BRD). Additionally, we may verify that the `photoprint` application implements those requirements correctly by performing a security code review or a penetration test.

## Design

The classic ```photoprint``` web application uses the Authorization Code Grant. This flow consists of the following steps.

![Authorization Code Grant](./pics/AuthorizationCodeGrant.png)

1. A user, let's call her Vivian, navigates to the printing website. This website is called the ***`Client`***. Vivian uploaded the pictures to picture gallery site (Gallery). The printing website (client, Print) offers the possibility to obtain pictures from the gallery site via a button that says *“Print pictures from the gallery site”*. Vivian clicks that button.
2. The client redirects her to an Authorization Server (AS; Authorization Endpoint). That server allows Vivian to authenticate to the gallery site and asks her if she consents to the Client accessing her pictures.
3. Assuming that Vivian gives her consent, the AS generates an Authorization Code (Authorization Grant) and sends it back to Vivian’s browser with a redirect command toward the return URL specified by the Client. The Authorization Code is part of that URL.
4. The browser honors the redirect and passes the Authorization Code to the Client.
5. The Client forwards that Authorization Code together with its own credentials to the AS (Token Endpoint). From this step on, the interactions are server-to-server. The Authorization Code proves that Vivian consented to the actions that the Client wants to do. Moreover, the message contains the Client’s own credentials (the Client ID and the Client Secret).
6. Assuming that the Client is allowed to make requests, the AS issues to the Client an Access Token, which can be used to make calls to the API of the gallery site (which offer access to Vivian’s pictures). Vivian is the Resource Owner, her pictures are Protected Resources (PRs), while the gallery site is the Resource Server (RS). The AS may also issue a Refresh Token. The refresh token enables the Client to obtain new access tokens; e.g. when the old ones expire.
7. The Client can finally access the PR; Vivian's pictures.

Besides selecting the authorization code grant for our classic web application, we made the following design decisions:

- we allow the usage of refresh tokens as we want to avoid that users need to authenticate every time the access tokens expire.
- we use custom handle-based tokens that are added as a Bearer header as we do not like JWTs.
- we combine the Authorization Server and Resource Server into one as we have a simple application.

## Insecure Implementation

### Gallery

We decide to implement our Gallery server with the [MEAN stack](http://mean.io/) : our server runs in [`express.js`](https://expressjs.com/) on top of  [`node.js`](https://nodejs.org/en/) and uses [`MongoDB`](https://www.mongodb.com/) as database. Feel free to skip this Section if you are not interested in implementation details. Read the [introductions](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Express_Nodejs/Introduction) on the [Mozilla website](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Express_Nodejs/Tutorial_local_library_website) to [understand](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Express_Nodejs/skeleton_website) how to create a basic node.js application.

Our gallery application is structured like a typical MEAN stack application:

- it implements the ***[Model-View-Controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)*** pattern. The model is the central component that manages the data, logic, and rules of the application. The view is the component that generates the output to the user based on changes in the model; i.e. it consists of the pages/responses that we are going to send to the Client. The controller forms the glue between models and views: it queries the models for data and puts that data into views that is sent to clients (users). The model and controller are custom code, while for the views we use the default [`Pug` (`Jade`)](https://github.com/pugjs/pug) as view template engine.
- Besides the MVC pattern, express.js applications also use a ***[router](https://expressjs.com/en/guide/routing.html)*** that maps URIs to controller functions. Architecturally speaking, this may be part of the controller, but most express.js applications use a separate folder for that.  

The flow throughout our express.js gallery application is as follows. A *Client* makes an HTTP request to our gallery application. The express.js enabled gallery application first passes the request throughout various middleware functionality (plugins that extend the functionality of an express.js application) and then passes it to a route handler. The route handler parses the URI and gives the URI parameters to the implementation(s) associated with that route,  typically a call (in our controller) to our model, a call to a middleware function, or a list of calls to middleware functions or custom code. Once a response is ready, the result is given to the view engine. This engine renders the response that is given to the *Client*.

![Our Gallery Application is an Express.js app](./pics/ExpressMVC.png)

The source code of our Gallery application is structured like a regular express.js application. We created a  general ```app.js``` script that combines the main server. The routes that map URIs to its actual implementation are defined in the ```routes``` folder, the controllers are defined in the controllers folder, the models are defined in the models folder, and the views are defined in the views folder. The `package.json` file defines the application dependencies and other information, while the public folder defines any stylesheets, images, and third-party JavaScript libraries. The structure is thus as follows.

```bash
/gallery
    app.js
    package.json
    /controllers
    /models
    /node_modules
    /public
        /images
        /javascripts
        /stylesheets
            style.css
    /routes
        index.js
        users.js
    /views
        error.jade
        index.jade
        layout.jade
```

We decided to use the following modules (middleware):

- ***[express](https://github.com/expressjs)*** to create an express application.
- ***[mongoose](https://github.com/Automattic/mongoose)*** to store our models in a MongoDB database.
- ***[oauth2orize](https://github.com/jaredhanson/oauth2orize)*** to make our server OAuth 2.0 aware.
- ***[morgan](https://github.com/expressjs/morgan)*** to log messages.
- ***[express-session](https://github.com/expressjs/session)*** to maintain sessions.
- ***[body-parser](https://github.com/expressjs/body-parser)*** to parse request bodies.
- ***[errorhandler](https://github.com/expressjs/errorhandler)*** to handle errors during development.

We could have used alternative modules that offer the same functionality, but the above packages are typically used within an express.js application.

The main API of our gallery application is fairly simple. We offer an API for manipulating user profiles and one for manipulating a user's gallery. The API to manipulate a user profile consists of a GET, PUT, and DELETE against a URI with the unique username. These operations respectively get the user profile, modify the profile, or delete the profile. A POST against the main user route creates a new user.

```javascript
router.get('/users/:name', ... );
router.put('/users/:name', ...);
router.delete('/users/:name', ...);
router.post('/users', ...)
```

The API to manipulate a user's gallery consists of a GET and POST against the main gallery URI as well as a GET, POST, DELETE, and PUT against the URI of a specific picture. The GET and POST against the main gallery URI lists the meta-data of the uploaded pictures or creates a new picture respectively. The GET, PUT, and DELETE requests against a specific picture obtain that picture, update the meta-data of that picture, or delete that picture respectively.

```javascript
router.post('/photos', ...);
router.get('/photos',  ...);
router.get('/photos/:username', ...);
router.get('/photos/:username/:imageid/view', ...);
router.get('/photos/:username/:imageid', ...);
router.get('/photos/:username/:imageid/raw',...);
router.put('/photos/:username/:imageid', ...);
router.delete('/photos/:username/:imageid', ...);
```

We use oauth2orize to offer the above API towards OAuth 2.0 clients. To use oauth2orize, we include the library in the code (with `require`) and instantiate it with `createServer`. We register callback functions in this server. Our callback functions contain code to

- generate an authorization code. Our function `grantcode` is registered as a callback to `oauth2orize.grant.code`.
- exchange an authorization code for an access token. Our function `exchangecode` is registered as a callback to `oauth2orize.exhange.authorizationCode`.
- (de-)serialize clients. Our functions `serialize` and `deserialize` are registered to transform a client object into a `client_id` and obtain a client object given the `client_id`.

```javascript
//import the oauth2orize package
var oauth2orize = require('oauth2orize');
// create OAuth 2.0 server
var server = oauth2orize.createServer();
// Register serialialization and deserialization functions.
server.serializeClient(serialize);
server.deserializeClient(deserialize);
// Register supported grant types.
server.grant(oauth2orize.grant.code(grantcode)
);
// Exchange authorization codes for access tokens.
server.exchange(oauth2orize.exchange.authorizationCode(exchangecode));

// function to generate an authorization code
function grantcode() {...}
//function to exchange an authorization code for an access token
function exchangecode() {...}
//function to serialize a client
function serialize() {...}
//function to deserialize a client
function deserialize() {...}
```

These functions are wired into our API. The remainder of this section walks you through the OAuth 2.0 Authorization Grant again and explains how we made our API Oauth 2.0 aware with `oauth2orize`.

#### Step 1: Click print Button

Vivian authenticates to the ```photoprint``` website and clicks the 'Print Pictures From Gallery' button.

![The Print Button on photoprint.](./pics/printpicturesfromgallery.png)

#### Step 2: Redirect to Gallery, Authenticate, and Consent Request

After Vivian clicked the print button, she is redirected by the Client (photoprint) to the Authorization Endpoint on the Authorization Server. That redirection request is as follows.

```http
GET /oauth/authorize?redirect_uri=http%3A%2F%2Fphotoprint%3A3000%2Fcallback&scope=view_gallery&response_type=code&
client_id=print HTTP/1.1
Host: gallery:3005
Connection: keep-alive
Referer: http://photoprint:3000/
```

As you notice, the URI contains the parameters `redirect_uri`, `scope`, `response_type`, and `client_id`. The `redirect_uri` is where `oauth2orize` will redirect Vivian after having created an authorization code. The `scope` is the access level that the client needs (`readgallery` is a custom scope that enables clients to read the pictures from a user's gallery). The `response_type` is code as we want an authorization code. The `client_id` is an identifier that represents the `photoprint` application.

During processing of the request above, the server must authenticate the user and ask for their permission to create an authorization code for the Client. The code below implements this functionality in the express route authorize. The implementation first authenticates a user via the connect-ensure-login middleware (line 3). After authentication, the endpoint calls the authorization function offered by oauth2orize. This function has two callbacks, namely `authorization_validate` and `authorization_autoapprove`. The `authorization_validate` callback should verify whether the client making the request is allowed to do so. The `authorization_autoapprove` function checks whether the client is trusted or whether the user had approved the client before, and if so automatically grants an authorization code. If that was not the case, a dialog is rendered. The dialog asks the user for approval.

```javascript
// user authorization endpoint
router.get('/authorize',
 login.ensureLoggedIn(),
 server.authorization(authorization_validate, authorization_autoapprove),
 renderdialog
)

function authorization_validate(clientID,redirectURI, done) {
  ...
}
function authorization_autoapprove(client,user, done) {
  ...
}
function renderdialog(req, res){
 res.render('dialog', {
    transactionID: req.oauth2.transactionID,
    user: req.user,
    client: req.oauth2.client
  });
}

```

In our example, the dialog is rendered as the user had not approved the client before and the client
is not trusted.

![The user needs to approve photoprint to access the pictures at gallery.](./pics/authcodegrant-dialog.png)

#### Step 3: Approval and Generation of Authorization Code

If the user clicks the Allow button, she submits the following request to the decision endpoint. Note that the transaction id is generated by oauth2orize and hard to guess.

```http
POST /oauth/authorize/decision HTTP/1.1
Host: gallery:3005
Connection: keep-alive
Content-Length: 23
Origin: http://gallery:3005
Referer: http://gallery:3005/oauth/authorize?redirect_uri=http%3A%2F%2Fphotoprint%3A3000%2Fcallback&scope=view_gallery&response_type=code&client_id=print

 'transaction_id=ATkQA32i'
```

The submission is processed by the endpoint. This function endpoint ensures that decision the user is authenticated and then invokes the decision function of oauth2orize. This decision function processes a user's decision to allow or deny access requested by a client application.

```javascript
router.post('/authorize/decision',
 login.ensureLoggedIn(),
 server.decision()
);

```

Based on the grant type requested by the client (the `response_type` parameter of the original request), the code grant function will be invoked to send an *Authorization Code* to the client.  The grant function takes the `client` requesting authorization, the `redirectURI` (which is used as a verifier in the subsequent exchange), the authenticated `user` granting access, their optional response (which contains approved scope, duration, etc. as parsed by the application), and an optional request (not listed below).  The generated code is given back to oauth2orize via the done callback. oauth2orize will in its turn send the code via a redirect to the client.

```javascript
server.grant(oauth2orize.grant.code(grantcode));

function grantcode(client, redirectURI, user, response, done) {
 var code = ...;
 <snip>
 done(null, code);
}
```

#### Step 4: Redirection To The Client

The server redirects the user back to the `photoprint` application (using the original `redirectURI` parameter) with an authorization code in the URL.

```http
GET /callback?code=92890 HTTP/1.1
Host: photoprint:3000
Connection: keep-alive
Referer: http://gallery:3005/oauth/authorize?redirect_uri=http%3A%2F%2Fphotoprint%3A3000%2Fcallback&
scope=view_gallery&response_type=code&client_id=print
```

#### Step 5:  Client Sends Authorization Code And Its Credentials To The Token Endpoint

The `photoprint` application then exchanges the *Authorization Code* and its *OAuth Client Id and Secret* for an *Access Token* by invoking the token endpoint. The request contains the parameters `authorization_code`, `redirect_uri`, `grant_type`, `client_id`, and `client_secret`. The `authorization_code` is a proof that the user approved this client to access their resources. The `grant_type` is the type of the code that was delivered (i.e., an `authorization_code`). The `redirect_uri` is the URI where the *Access Tokens* will be delivered. The `client_id` and the `client_secret` authenticate the client. They can be delivered via an Authorization header or as parameters in the body of the request.

```http
POST /oauth/token HTTP/1.1
Accept: application/json
host: gallery:3005
content-type:
application/x-www-form-urlencoded
content-length: 147
Connection: close
code=92890&redirect_uri=http%3A%2F%
2Fphotoprint%3A3000%2Fcallback&
grant_type=authorization_code&client_id=print
&client_secret=secret
```

#### Step 6: Issue An Access Token At The Token Endpoint

The token endpoint typically uses `Passport` strategies (an authentication middleware) to authenticate the client via either an HTTP Passport Basic authentication header (as provided by `passport-http`) or client credentials in the request body passport-http (as provided by `passport-oauth2-client-password`). The token endpoint is implemented by oauth2orize in the `server.token()` function. If an error occurs, the `errorHandler` function will    format an error response.  

```javascript
app.post('/token',
  passport.authenticate(['basic','oauth2-client-password'], { session: false}),
  server.token(),
  server.errorHandler() );
```

Based on the grant type issued to the client, oauth2orize invokes the appropriate exchange module (`oauth2orize.exchange.code`) to issue an access token. The `oauth2orize.exchange.code` function is a callback function that requires a callback implemented by the developer that performs the actual exchange. It takes as parameters the client, the authorization code, a redirect URI, and a callback parameter (`done`). The developers invoke the callback parameter to give the generated code to done `oauth2orize`. `oauth2orize` will then send the token to the client via a redirect.

```javascript
server.exchange(oauth2orize.exchange.authorizationCode(exchangecode));
function exchangecode(client, code, redirectURI, done) {
  <snip>
  var token = ...
  done(null, token, null, {'expires_in': 3600});
}
```

The server responds with an access token that the `photoprint` application can use to obtain a resource from the Gallery Application (Resource Server).

```http
X-Powered-By: Express
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
Connection: close
Content-Length: 64

{"access_token":"54698","expires_in":3600,"token_type":"Bearer"}
```

#### Step 7: Access a Resource

The print application then uses the access token to access a resource on gallery. In the example below, it accesses all the photos of a given user.

```http
GET /photos/me?access_token=54698 HTTP/1.1
Accept: application/json
Content-Length: 0
Host: gallery:3005
Connection: close
```

The gallery application then validates the access token and processes the request. Validation of the access token is outside of the scope of `oauth2orize`, but is typically done via a passport module or a custom function. We opted to use the `BearerStrategy` of passport, as our server issues Bearer tokens.

```javascript
passport.use(new BearerStrategy(
  function(accessToken, done) {
    AccessToken.findOne({token:accessToken}, function(err, token) {
      <snip>
    }); }));
```

### Print

Incidentally, the `photoprint` web application also uses the MEAN stack. The print application is a fairly simple application.
TODO implement obtaining a profile, authenticating, and storing orders (to illustrate OpenId connect).

## Security Considerations

In this section, we present common security mistakes made when designing/implementing an OAuth 2.0 enabled application. This section lists a subset of what is listed in [RFC 6819](https://tools.ietf.org/html/rfc6819).

### Gallery Authorization Server

#### Authorization Endpoint: Validate the RedirectURI Parameter

If the authorization server does not validate that the redirect URI belongs to the client, it is susceptible to two types of attacks:

- Open Redirect. ![Attacker redirects the victim the victim to a random site.](./pics/openredirect.gif)
- Stealing of Authorization Codes. ![Attacker steals a valid authorization code from the victim.](./pics/openredirect_stealauthzcode.gif)

To remediate this, validate whether the redirect_uri parameter is one the client provided during the registration process.

To validate this as a tester, change the value of the redirect_uri parameter to one you control.

```html
http://gallery:3005/oauth/authorize?response_type=code&redirect_uri=http%3A%2F%2Fattacker%3A1337%2Fcallback&scope=email&client_id=photoprint
```

#### Authorization Endpoint: Generate Strong Authorization Codes

If the tokens are weak, an attacker may be able to guess them at the token endpoint. This is especially true if the client secret is compromised, not used, or not validated.

To remediate this, generate authorization codes with a length of at least 128 bit using a secure pseudo-random number generator that is seeded properly. Most mature OAuth 2.0 frameworks implement this correctly.

To validate this as a tester, analyze the entropy of multiple captured authorization codes.

#### Authorization Endpoint: Expire Unused Authorization Codes

TODO

#### Token Endpoint: Invalidate Authorization Codes After Use

TODO

#### Token Endpoint: Bind the Authorization Code to the Client

TODO

#### Token Endpoint: Expire Access and Refresh Tokens

TODO

#### Token Endpoint: Generate Strong Handle-Based Access and Refresh Tokens

TODO

#### Token Endpoint: Store Handle-Based Access and Refresh Tokens Securely

TODO

#### Token Endpoint: Limit Token Scope

TODO

#### Token Endpoint: Store Client Secrets Securely

TODO

#### Token Endpoint: Bind Refresh Token to Client

TODO

#### Resource Server: Reject Revoked Tokens

TODO

#### Implement Rate-Limiting

TODO

### Photoprint OAuth 2.0 Client

#### Store Client Secrets Securely

TODO

#### Store Access and Refresh Tokens Securely

TODO

# Mobile Application: Authorization Code Grant with PKCE

The proof key for code exchange TODO [https://tools.ietf.org/html/rfc7636](https://tools.ietf.org/html/rfc7636)

## Design of OAuth 2.0 Mobile Application

TODO design content

## Implementation

RFC 6819 elaborates on common security mistakes within OAuth 2.0.

## Testing

TODO

# Single Page Application: Implicit Grant Flow

## Design of OAuth 2.0 SPA Client

TODO pic.

The implicit grant is a simplified authorization code flow in which the client
is issued an access token directly at the authorization endpoint, rather than
an authorization code. In our running example, it would look as follows.

1. Our user Vivian navigates to the printing website. This website is called the “Client”. Vivian uploaded the pictures to picture gallery site (Gallery). The printing website (client, photoprint) offers the possibility to obtain pictures from the gallery site via a button that says “Print pictures from the gallery site”. Vivian clicks that button.
2. The client redirects her to an Authorization Server (AS; Authorization Endpoint). That server allows Vivian to authenticate to the gallery site and ask her if she consents to the Client accessing her pictures.
3. Assuming that Vivian gives her consent, the AS generates an Access Token and sends it back to Vivian’s browser with a redirect command toward the return URL specified by the Client. The Access Token is part of that URL.
4. The browser honors the redirect and passes the Access Token to the Client. The Client can access the PR; Vivian's pictures with the Access Token.

## Implementation of OAuth 2.0 SPA Client

RFC 6819 elaborates on common security mistakes within OAuth 2.0.

## Testing of OAuth 2.0 SPA Client

TODO testing

# First Party Mobile Application: Resource Owner Password Credentials Flow

## Design of First Party Mobile Application Client

The resource owner password credentials grant is a simplified flow in which the client uses the
resource owner password credentials (username and password) to obtain an access token. In our
running example, it would look as follows.

TODO pic

1. A user, let's call her Vivian, navigates to the printing website. This website is called the “Client”. Vivian uploaded the pictures to picture gallery site (Gallery). The printing website (client, photoprint) offers the possibility to obtain pictures from the gallery site via a button that says “Print pictures from the gallery site”. Vivian clicks that button and provides her credentials.
1. The client submits Vivian's credentials to the Authorization Server.
1. The AS generates an Access Token and sends it back to the Client.
1. The Client can access the Resource Server; Vivian's pictures with the Access Token

## Implementation of First Party Mobile Application Client

RFC 6819 elaborates on common security mistakes within OAuth 2.0.

## Security Considerations of First Party Mobile Application Client

TODO security considerations

### OpenID Connect

Moreover, OAuth is the foundation for the single sign-on protocol OpenID Connect. OpenID
Connect builds upon OAuth and provides clearly defined interfaces for user
authentication and additional (optional) features, such as dynamic identity
provider discovery and relying party registration, signing and encryption of
messages, and logout.

# Checklists

## For Architects

The following questions obtain the context that is required to analyze the application for bad OAuth related design decisions.

- What is the client using the API?
  - [ ] Classic Web Application
  - [ ] Native Mobile Application
  - [ ] Mobile Application with Web View
  - [ ] Single Page Application
  - [ ] Thick client
  - [ ] Embedded Application
  - [ ] Other
- What token types does the application use?
  - [ ] Self-contained (e.g. JWTs)
  - [ ] Custom (handle-based e.g. unique ID)
  - [ ] WS-security
- What token profiles does the application use?
  - [ ] Bearer tokens
  - [ ] Holder of Key Tokens
- Who developed the client?
  - [ ] We (first-party)
  - [ ] Someone else (third party)

The checklist itself is as follows. Checked boxes are OK.

Use the following tree to determine whether the correct flow was chosen.
![A decision tree that helps an architect to decide which OAuth 2.0 flows to support](./pics/DecisionTreeFlow.png)

- [ ] How does the Client application authenticate users?
  - [x] OpenID Connect
  - [x] SAML Token as part of OAuth flow (SAML Extensions)
  - [ ] It assumes that the user is authenticated because it receives an OAuth token
- How does the client store OAuth client secrets and OAuth refresh tokens?
  - [ ] Hardcoded
  - [ ] Configuration file
  - [x] Using platform provided secure storage such as keychain or keystore on mobile or encrypted configuration files on .NET
- How does the client store access tokens?
  - [x] It does not store them
  - [ ] Hardcoded
  - [ ] Configuration file
  - [ ] Using platform provided secure storage such as keychain or keystore on mobile or encrypted configuration files on .NET
- How does the server store OAuth client secrets?
  - [ ] Hardcoded
  - [ ] Configuration file
  - [ ] Plain-text in Database
  - [x] Hashed in database (like user passwords)
  - [ ] Other
- If the server uses handle-based tokens, how does it store them?
  - [ ] Plain-text in a database
  - [x] Hashed
  - [x] Hashed and salted (this is not necessary, tokens are usually sufficiently long)
- If the server uses JWTs, what type of signatures does it use?
  - [ ] None
  - [x] RSA
  - [ ] Shared secret
- Does the server implement rate limiting against the endpoint to obtain tokens?
  - [x] Yes
  - [ ] No
- When do access tokens expire?
  - [ ] Never
  - [x] Within less than 30 minutes
  - [ ] Within a day
  - [ ] Longer than a day
- When do refresh tokens expire?
  - [ ] Never
  - [x] Within 30 days
  - [x] If mobile application with low risk profile, within 1 year
  - [ ] Other

## For Software Engineers

TODO

## For Testers

TODO

# Conclusion

In this article, we showed how to use OAuth 2.0 securely when using

- a [Classic Web Application](#classic-web-application-authorization-code-grant-flow),
- a [Single Page Application](#single-page-application-implicit-grant-flow), and
- a [Mobile Application](#mobile-application-authorization-code-grant-with-pkce) as clients. For each of these clients, we  elaborated on the overall design, implemented that design using the MEAN stack, and touched upon common security mistakes.
