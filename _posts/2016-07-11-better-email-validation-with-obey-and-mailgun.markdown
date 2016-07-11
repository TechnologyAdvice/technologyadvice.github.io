---
layout: post
title: "Better Email Validation with Obey and Mailgun"
date: 2016-07-11 10:00:00
categories: asynchronous validation nodejs
author: kent.safranski
cover: /assets/images/covers/mail-sorting.jpg
description: Email validation seems simple, but you really should be using a service to accurately validate addresses. Here's how Obey and Mailgun can work together to reliably validate emails.
comments: true
---

Validating email is simple, right? Just throw a regex at it? Well it's not as simple as you might think. The RFC Spec (Page 27 of [this monster](https://www.ietf.org/rfc/rfc0822.txt), if you're interested) is clear as mud on what constitutes a valid email.

With all the special characters and conditions of what can go before and after the `@`, a regex to validate emails accurately would be incomprehensable at best.

_If you'd like to find out more about why email validation sucks check out [this article](http://www.kbedell.com/2011/03/16/how-to-validate-an-email-address-using-regular-expressions/), for now let's move along._

Surely this is something other people have dealt with though, what about a library? Well, they [do exist](https://www.npmjs.com/package/isemail), however, we want to _really_ validate an email. This is most likely either a core communication or authentication property. Libraries can do more than a regex, but many lack more advanced checks against actual resources, looking at email services and their own history of valid domains/emails.

### Using a Service for Validation

We want to know if an email is valid, but regex is tricky and libraries are limited. This is where validation services come in handy. [Mailgun's API for email validation](https://documentation.mailgun.com/api-email-validation.html#email-validation) is a wonderful option. It will check a number of things for us:

* General syntax based on RFC
* DNS validation
* Spell check (common email services)
* Service Provider checking

Not only will it take care of the general syntax (what we _would_ use a hyper-complex regex for) but it will make "best-attempt" tries at checking for common errors and valid domains.

So, you may be asking yourself what this looks like. Typically validation is a synchronous task but we're going to call an API to make this work. This is where [Obey](https://github.com/TechnologyAdvice/obey) makes things simple. The promise-first design allows it to handle async and sync requests exactly the same. But what does this look like?

We start with a simple type definition. Using Obey and [request-promise](https://www.npmjs.com/package/request-promise) we'll hit the API and handle the results:

```javascript
const obey = require('obey')
const rp = require('request-promise')

// Our API key
const mailgunPublicKey = 'XXXXX-YYYY-ZZZZZZZ'

obey.type('mailgunEmail', (context) => {
    // Hit the API
    return rp({ 
        uri: `https://api:${mailgunPublicKey}'@api.mailgun.net/v3/address/validate`,
        qs: {
            address: context.value
        },
        json: true 
    }).then((result) => {
        // Handle validation errors
        if (result.is_valid) {
            context.fail(context.value + ' is an invalid email')
        }
    })
})
```

Now when we define a model or rule we can use the `mailgunEmail` type and validation against the Mailgun API will occur. If the address returns invalid it will call `context.fail` resulting in a `ValidationError`.

### Expanding the Concept

The above is a fairly straightforward example. However, Mailgun isn't your only option for email validation, and there are a myriad great validation APIs for other data types like phone numbers and addresses. The code could easily be modified to perform all sorts of asynchronous validation.

### Get the Plugin

If you're just interested in Mailgun we have an [`obey-type-email-mailgun`](https://github.com/TechnologyAdvice/obey-type-email-mailgun) plugin all ready for install and use.
