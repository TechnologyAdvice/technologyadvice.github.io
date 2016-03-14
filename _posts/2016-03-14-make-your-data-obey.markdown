---
layout: post
title: "Make Your Data Obey"
date: 2016-03-14 06:00:00
categories: obey data validation modelling
author: kent.safranski
cover: /assets/images/covers/obey.jpg
description: Consistent data is key to any successful application. Find out how the Obey library can help you model and validate your data more efficiently.
comments: true
disqus: true
---

Data is what programming is all about. Applications would be nothing without that beautiful i/o of data moving between modules, services, and data storage. But data is nothing if it's not consistent.

> [Obey](https://github.com/TechnologyAdvice/obey) was developed specifically to address, and provide solutions for, data modelling and validation for modern web applications developed in JavaScript.

## Diving In

Let's start with a simple example: validating a user account. There are some core data points we want to ensure, using Obey's `model` method to create a model object is how we start:

{% highlight javascript %}
import obey from 'obey'

const user = obey.model({
    id: { type: 'uuid', required: true },
    email: { type: 'email', required: true },
    password: { type: 'string', required: true }
})
{% endhighlight %}

The above should all be fairly easy to understand. Now let's put it into action, using the `validate` method, passing the data object, then saving to a datasource. Let's assume this is going into a datasource using a CRUD library, already setup, called `db`. Using the model above:

{% highlight javascript %}
import db from './db'

user.validate({
    id: 'fe6f80c6-c530-4f10-ad0c-89f0f68a879f',
    email: 'jdoe@email.com',
    password: 'p@55w0rd'
})
// Assumes a db.create method which accepts an object as argument
// and returns a promise
.then(db.create).then(res => {
    console.log('User Created!', res)
})
.catch(err => {
    console.log('Failed!', err)
})
{% endhighlight %}

Overall pretty simple stuff. We validate the model, then create the user (or catch the error). But what about that `id`? We probably want to have that generated automatically.

## Creators

Obey supplies `creators` for generating data using methods that can be predefined. For the `id` we'll use the [node-uuid](https://www.npmjs.com/package/node-uuid) module. Let's revisit the model, defining a `creator` using the method by the same name:

{% highlight javascript %}
import obey from 'obey'
// Import the node-uuid lib
import uuid from 'node-uuid'

// Add a creator that returns a v4 UUID if one isn't supplied
obey.creator('uuidCreator', uuid.v4)

const user = obey.model({
    // The id now specifies that the creator 'uuid' should be 
    // used if no value is present
    id: { type: 'uuid', creator: 'uuidCreator' required: true },
    email: { type: 'email', required: true },
    password: { type: 'string', required: true }
})
{% endhighlight %}

So, now we just pass in the `email` and `password` and our `id` is auto-generated.

But that `password` is not cool; we don't want to save that in plain-text...

## Modifiers

Obey wasn't just built for validation, but also to support coercion and modification. Modifiers allow for doing this in a clean, simple way. Let's revisit the model again, using the `modifier` method to create one:

{% highlight javascript %}
import obey from 'obey'
import uuid from 'node-uuid'
// Import the argon2 module
import argon2 from 'argon2'

obey.creator('uuidCreator', uuid.v4)

// Add a modifier that returns md5 encrypted value
const salt = new Buffer('somesalt')
obey.modifier('encrypt', val => argon2.hash(val, salt))

const user = obey.model({
    id: { type: 'uuid', creator: 'uuidCreator' required: true },
    email: { type: 'email', required: true },
    // Password now specifies the modifier 'encrypt' should be used to
    // encrypt the password
    password: { type: 'string', modifier: 'encrypt', required: true }
})
{% endhighlight %}

Now not only does our validation method create a UUID, it also encrypts the password for us.

But what about that `email` property? Probably want to make sure it's unique...

## Types

Having that `email` property contain one that's already in the datasource would be bad news, right? Some databases do provide uniqueness integrity checking, but there are a lot that don't. Luckily, `types` are extensible with Obey, so we can add our own check:

{% highlight javascript %}
import obey from 'obey'
import uuid from 'node-uuid'
import argon2 from 'argon2'
// Here's our db reference again...
import db from './db'

obey.creator('uuidCreator', uuid.v4)

const salt = new Buffer('somesalt')
obey.modifier('encrypt', val => argon2.hash(val, salt))

// Add a type to ensure we have a valid, unique email
obey.type('uniqueEmail', context => {
    // Read from the 'db' datasource where email is the value
    return db.read({ email: context.val }).then(res => {
        // Fail if it already exists
        if (res.length !== 0) context.fail('Email must be unique')
        // Note: you'd also want to check that the email is valid!
    })
})

const user = obey.model({
    id: { type: 'uuid', creator: 'uuidCreator' required: true },
    // Email is now set to type 'uniqueEmail'
    email: { type: 'uniqueEmail', required: true },
    password: { type: 'string', modifier: 'encrypt', required: true }
})
{% endhighlight %}

Now when the `validate` method is run we do a read on the datasource and check that the email doesn't already exist. Obey doesn't need to be returned a value or `true`, if the `context.fail` method is not called it will simply move forward, adding the error to a collection which is included with any other errors.

What's best is **we do all of this at the data validation level**. Yes, we're still hitting the datasource (really no way around that), but all of our handling of an error condition is in the same place as the rest of the validation error handling.

The code above is also getting a bit unweildy and is probably better abstracted, but Obey is built for that; all of the modifiers, creators, and types can be added in an abstraction then called wherever you create a model. This makes it easy to share types, modifiers, creators, and the like across multiple models. This can be done by simply moving the methods to a file, we'll call it `obey-utils.js`:

{% highlight javascript %}
import obey from 'obey'
import uuid from 'node-uuid'
import argon2 from 'argon2'
import db from './db'

obey.creator('uuidCreator', uuid.v4)

const salt = new Buffer('somesalt')
obey.modifier('encrypt', val => argon2.hash(val, salt))

obey.type('uniqueEmail', context => {
    return db.read({ email: context.val }).then(res => {
        if (res.length !== 0) context.fail('Email must be unique')
        // Note: you'd also want to check that the email is valid!
    })
})
{% endhighlight %}

But wait, that custom type is asynchronous? Why yes it is. The creators and modifiers can be as well...

## Embracing Asynchronicity

It's simple to look at data modelling and validation as a synchronous thing; running through checks and moving on, but it should have more to it than just that. Addressing modelling and validation only on a synchronous level limits the power of this integral layer of application development.

With so much in data i/o being asynchronous, modelling is one area where asynchronous processing is not a _nice to have_, it's a **must have**.

Let's take a look at our whole model now, with the `obey-utils` abstracted into a separate file:

{% highlight javascript %}
import obey from 'obey'
import db from './db'
import './obey-utils'

// Setting up the model
const user = obey.model({
    id: { type: 'uuid', creator: 'uuidCreator' required: true },
    email: { type: 'uniqueEmail', required: true },
    password: { type: 'string', modifier: 'encrypt', required: true }
})

// Validation
user.validate({
    email: 'jdoe@email.com',
    password: 'p@55w0rd'
})
.then(db.create).then(res => {
    console.log('User Created!', res)
})
.catch(err => {
    console.log('Failed!', err)
})
{% endhighlight %}

By embracing asynchronicity everything runs through in the same fashion, using the same conventions and either moving forward as intended or throwing to the `catch` and being handled cleanly.

## Summary

[Obey](https://github.com/TechnologyAdvice/obey) was developed to support an area of development that oftentimes doesn't get enough attention, or tackles validation by assuming a datasource will handle it for you.

We wrote [Obey](https://github.com/TechnologyAdvice/obey) to centralize our data validation and modelling in one place, stop relying on after-the-fact error handling and injecting validation logic into places it shouldn't be, but more than anything we wrote it to be flexible and easy to work with.

In addition to what is show above, Obey provides numerous other tools for modelling and validation:

* Reusable, JSON-based schemas
* Global validators such as `min`, `max`, `allow` and `default` which work with all types
* Recursive types
* Sub-type support for multiple formats
* Individual rule definition
* `strict` mode for enabling/disabling exact object matching
* Highly extensible API

### Your Feedback

While we've been dogfooding Obey and working out the pain points _we_ saw in modelling and validation, we want to know what you see as issues or challenges commonly faced when working with data.

What areas have you felt other, similar libraries have been good or bad at addressing, and do you see things missing from Obey that we can add?
