---
published: true
title: Writing an OAuth 2 Handler
layout: post
---
So, I was encouraged to write some content (a couple of paragraphs) about putting together an OAuth handler for Meteor. Now, Meteor provides a number of ready-to-use handlers for some of the "big names", but there's no real help if you want to use your own service, or some other 3rd party provider.

Sadly, I blew the two paragraph target, but I hope this all proves useful to someone!

## What is OAuth?

OAuth is a way of providing access to third party application resources (such as Facebook) from a user written application (such as a Meteor app). For example, we can use Facebook's OAuth service to enable our Meteor app to gain access to a user's email address.

OAuth is an *authorization* service, rather than *authentication* service. Nonetheless, it's common practice to use OAuth as a means of authenticating users to your application.

There are advantages to this:

- We have a reasonable expectation that the account details for Facebook, Google, Twitter, etc., have been secured using industry best practices. As long as we are careful to protect private data (such as our application's *secret key*), we can leverage their investment in security to our advantage.
- We only need to remember one set of credentials (at least in theory).
- It's an easy way to equip an app, or multiple apps with the appearance of seamless connectivity across various devices.

The disadvantages are that if (for example) Facebook's OAuth service is your only means of authenticating your users, then you are constraining your users to have a Facebook account, and you are reliant on Facebook not changing their service in such a way as to affect their ability to use your app. In Meteor it's simple to have password authentication as a fallback, or to allow multiple OAuth services to be available.

## OAuth Versions

OAuth 1.0: The [original protocol](http://tools.ietf.org/html/rfc5849). Complex to implement, but generally considered more secure than OAuth 2.0.

OAuth 2.0: The protocol in use by most "big name" providers. Relatively simple to implement, but may be less secure than OAuth 1.0.

## Writing an OAuth client for Meteor

This article concentrates on implementing a popular (Imgur) OAuth 2 login handler. If you want to use it as a base for writing your own, whether for Imgur or something else, I've tried to annotate the code well, especially where it looks complicated. In practice, it's largely boilerplate based off Meteor's own Google OAuth handler.

### Understand the flows and data

Even though Meteor abstracts some of the complication around OAuth flows, understanding the sequence of events, as well as the data in and out, for the OAuth service you want to write for is important. The flows are based around standard HTTP(S) POST and REDIRECT methods.

Spring have some great flow diagrams, for [OAuth 1](http://docs.spring.io/spring-social/docs/1.1.0.RELEASE/reference/htmlsingle/#section_oauth1ServiceProviders) and [OAuth 2](http://docs.spring.io/spring-social/docs/1.1.0.RELEASE/reference/htmlsingle/#section_oauth2ServiceProviders). It's worth spending a little time becoming acquainted with how OAuth works. 

Data is typically passed as query parameters in the URI (REDIRECT/GET), or as POST data. Returned data is commonly encoded as JSON.

You need solid documentation from your OAuth provider if you intend writing a Meteor OAuth handler.

### The Meteor way

Meteor provides us with some underlying code to manage the flows and help us write an OAuth 1.0 or OAuth 2.0 client. We just need to follow some simple guidelines to ensure that everything is kept in step.

For any given provider 'xxx', we write two packages:

- xxx
  - Provides information for the configuration dialog.
  - Defines the namespace (export).
  - May have a whitelist of required resources.
  - Registers 'xxx' as an OAuth provider.
  - Provides the functionality for the flow logic.
  - Gets resources from (multiple) endpoint(s).
- accounts-xxx
  - Provides the Meteor accounts tie-in.
  - Defines the `loginButtons` helper.
  - Defines the `loginButtons` logo for 'xxx'.

Let's look at what's involved. We'll write an OAuth handler for [Imgur](https://api.imgur.com/oauth2). In order to test or use this we will need to register an app with Imgur to get a `client_id` and `client_secret`. We use these in conjunction with the documented endpoints and data structures to write our handler.

I chose Imgur because its OAuth service is well documented, although a certain amount of "hopping around" was needed to get all the information I needed. You will almost inevitably go through this process for whatever service you write for.

You will find I have used the term "boilerplate" in a lot of places (here and in the code). By this I mean that it's fundamentally unchanged for any OAuth 2 handler. Often, little more than identifying and changing the service name is needed. So, starting from my code, you would replace 'Imgur' with 'Xxx' and 'imgur' with 'xxx' wherever you see it. Clear exceptions to this rule of thumb will be endpoints and the data they return, which will be very provider-specific.

Note that rather than go through the code step-by-step in this article, I have provided some background information and links to the Github repo(s), so you can look at the code in context.

Finally, I wrote my packages using ES2015 syntax (where I have remembered to) - I ensured the `ecmascript` package was specified in the `package.js` files.

### imgur package

Within the `imgur` package we set up five files for the handler, along with files for any unit tests we may write. I have excluded tests in this list of files (and in the `package.js` file):

```
imgur_configure.html
imgur_configure.js
imgur_client.js
imgur_server.js
package.js
```

#### imgur_configure.html

I started with the configuration dialog. This comprises `imgur_configure.html` and `imgur_configure.js`. Meteor uses these files when you first attempt to authorize to the OAuth service, in order to display a prompted dialog which it uses to store the credentials applicable to that OAuth service.

`imgur_configure.html` contains a template named `configureLoginServiceDialogForImgur` and contains instructions for completing the dialog. The template name is standard boilerplate (`configureLoginServiceDialogForXxx`), and the template itself can be as detailed as you like. You can look at MDG's [google_configure.html](https://github.com/meteor/meteor/blob/devel/packages/google/google_configure.html) for inspiration. You can find the one I put together for Imgur [here](https://github.com/robfallows/tunguska-imgur/blob/master/imgur_configure.html).

#### imgur_configure.js

`imgur_configure.js` is fundamentally boilerplate and contains a list of fields used to populate the dialog form. It will always contain fields for the `clientId` and `secret` (corresponding to Imgur's `client_id` and `client_secret`). If your dialog requires other data to be entered during configuration, this is the place to do it. The one I wrote is [here](https://github.com/robfallows/tunguska-imgur/blob/master/imgur_configure.js).

#### imgur_client.js

The client component does little more than initiate the authorization request to the appropriate OAuth service endpoint. By convention this is an OAuth redirect flow to obtain a code (to be exchanged later for a token). The redirect is hooked into the `imgur_server.js` code via an automatically created server-side REST endpoint at `/_oauth/imgur`. Mine is [here](https://github.com/robfallows/tunguska-imgur/blob/master/imgur_client.js).

#### imgur_server.js

The server component is responsible for the initial OAuth flow to exchange a `code` for a `token`. Most of this is boilerplate (little more than ensuring references to the service name are correct and the right endpoints are accessed).

When the `imgur_client.js` executes the authorization request, Meteor's underlying OAuth code sets up a server-side REST endpoint (at `/_oauth/imgur`) which ensures the `retrieveCredential` method in this file is executed. This file is not particularly complex - it's just a careful sequence of boilerplate actions with some specifics for this service.

These are the steps we go through:

* Define the base object namespace (boilerplate).
* Define the retrieveCredential hook for use by underlying Meteor code (boilerplate).
* Define the fields we want (whitelist). Note that they come from various endpoints (boilerplate).
* Register this service with the underlying OAuth handler.
  * Make sure we have a config object (boilerplate).
  * Get the token (Meteor handles the underlying authorization flow) (mostly boilerplate).
  * Get required data from the account endpoints with the token. (service-specific)
  * Build the serviceData object (mostly boilerplate).
  * Return the serviceData object and options object (boilerplate).

My code for `imgur_server.js` is [here](https://github.com/robfallows/tunguska-imgur/blob/master/imgur_server.js).

#### package.js

This file needs to have all the dependencies added for this package. It's boilerplate stuff and you can see my file [here](https://github.com/robfallows/tunguska-imgur/blob/master/package.js).

### accounts-imgur package

Within the `accounts-imgur` package we set up three files for the package, along with files for any unit tests we may write. I have excluded tests in this list of files (and in the `package.js` file):

```
accounts-imgur.js
accounts-imgur_login_button.css
package.js
```

### accounts-imgur.js

Completely boilerplate - just ensure that `Imgur` or `imgur` is used in the right places. Mine's [here](https://github.com/robfallows/tunguska-accounts-imgur/blob/master/accounts-imgur.js).

### accounts-imgur_login_button.css

This is boilerplate in structure - just ensure that the class name has `imgur` in it. This file provides the logo which appears on the login button. It's a 16x16 URL-encoded `png`. Mine's [here](https://github.com/robfallows/tunguska-accounts-imgur/blob/master/accounts-imgur_login_button.css).

#### package.js

Boilerplate. Mine's [here](https://github.com/robfallows/tunguska-accounts-imgur/blob/master/package.js). Note that this package depends on `tunguska:imgur`, which means you only need to explicitly `meteor add tunguska-accounts-imgur` to get Imgur authorization in your app.

## What next?

- I've put both packages on atmosphere, if you want to try them. However, they're sufficiently reusable on any OAuth 2 service, so I encourage you to clone them and add to the ecosystem with your favorite service!
- Add tests!
- PR's welcome (especially tests!)