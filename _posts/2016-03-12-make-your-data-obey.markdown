---
layout: post
title: "Make Your Data Obey"
date: 2016-03-12 10:00:00
categories: obey data validation modelling
author: kent.safranski
cover: /assets/images/covers/obey.jpg
description: Data is what programming is all about. Applications would be nothing without that beautiful i/o of data moving between modules, services, and data storage. But data is nothing if it's not consistant.
comments: true
disqus: true
---

Data is what programming is all about. Applications would be nothing without that beautiful i/o of data moving between modules, services, and data storage. But data is nothing if it's not consistant.

**[Obey](https://github.com/TechnologyAdvice/obey) was developed specifically to address, and provide solutions for, data modelling and validation for modern web applications developed in JavaScript.**

## Diving In

Let's start with a simple example; validating a user account. There are some core data points we want to ensure, so a model designating these is a good starting point:

{% highlight javascript %}
import obey from 'obey'

const user = obey.model({
    id: { type: 'uuid', required: true },
    email: { type: 'email', required: true },
    password: { type: 'string', require: true }
})
{% endhighlight %}

The above should all be fairly easy to understand. Now let's put it into action, saving to a datasource. Let's pretend this is going into a datasource using a CRUD lib, already setup, called `db`. Using the model above:

{% highlight javascript %}
import db from './db'

user.validate({
    id: 'fe6f80c6-c530-4f10-ad0c-89f0f68a879f',
    email: 'jdoe@email.com',
    password: 'p@55w0rd'
})
// Assumes a db.create method which accepts an object as argument
.then(db.create).then(res => {
    console.log('User Created!', res)
})
.catch(err => {
    console.log('Failed!', err)
})
{% endhighlight %}

Overall pretty simple stuff. We validate the model, then create the user (or catch the error). But what about that `id`? We probably want to have that generated automatically.

## Creators

Obey supplies `creators` for generating data using methods that can be predefined. For the `id` we'll use the [node-uuid](https://www.npmjs.com/package/node-uuid) module. Let's revisit the model:

{% highlight javascript %}
import obey from 'obey'
// Import the node-uuid lib
import uuid from 'node-uuid'

// Add a creator that returns a v4 UUID if one isn't supplied
obey.creator('uuidCreator', uuid.v4)

const user = obey.model({
    // The id now specifies that the creator 'uuid' should be used if no value is present
    id: { type: 'uuid', creator: 'uuidCreator' required: true },
    email: { type: 'email', required: true },
    password: { type: 'string', require: true }
})
{% endhighlight %}

So, now we just pass in the `email` and `password` and our `id` is auto-generated.

But that `password` is still not cool; we don't want to save that in plain-text...

## Modifiers

Obey wasn't just built for validation, but also to support coercion and modification. Modifiers allow for doing this in a clean, simple method. Let's revisit the model again:

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
    // Password now specifies the modifier 'encrypt' should be used on the value
    password: { type: 'string', modifier: 'encrypt', require: true }
})
{% endhighlight %}

Alright, so now not only does our validation create a UUID, it also encrypts that password for us. We've changed the model to accomplish this, but we validate exactly the same, Obey handles this all automatically based on the modelling we defined in one location.

BUT, what about that `email`? Probably want to make sure it's unique...

## Types

Having that `email` contain one that's already in the datasource would be bad news bears right? Some databases do provide uniqueness integrity checking, but there are a lot that don't. Luckily, `types` are extensible with Obey, so we can add our own check:

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
        // Fail if it's not a legit email
        const emailRegEx = /^([a-zA-Z0-9_\.\-])+\@(([a-zA-Z0-9\-])+\.)+([a-zA-Z0-9]{2,4})+$/
        if (!context.value.match(emailRegEx)) context.fail('Must be a valid email')
    }
})

const user = obey.model({
    id: { type: 'uuid', creator: 'uuidCreator' required: true },
    // Email is now set to type 'uniqueEmail'
    email: { type: 'uniqueEmail', required: true },
    password: { type: 'string', modifier: 'encrypt', require: true }
})
{% endhighlight %}

Now when the `validate` method is run we do an asynchronous read on the datasource, check that the email 1) doesn't exist and 2) is a legit email address.

What's best is **we do all of this at the data validation level**. Yes, we're still hitting the datasource (really no way around that), but we're using a less expensive process and all of our handling of an error condition is in the same place as the rest of the validation error handling.

The code above is also getting a bit unweildy and is probably better abstracted, but Obey is built for that; all of the modifiers, creators, and types can be added in an abstraction then called wherever you create a model.

But wait, that custom type is asynchronous? Why yes it is. The creators and modifiers can be as well...

## Embracing Asynchronicity

It's simple to look at data modelling and validation as a synchronous thing, but it really shouldn't be. It limits the power of this important layer of application development.

With everything else in data i/o being asynchronous it's one area where asynchronous processing is not a _nice to have_, it's a **must have**.

Let's take a look at our whole build process now, abstracting the Obey modifier, creator and type into their own lib we'll call `obey-utils`.

{% highlight javascript %}
import obey from 'obey'
import db from './db'
import './obey-utils'

// Setting up the model
const user = obey.model({
    id: { type: 'uuid', creator: 'uuidCreator' required: true },
    email: { type: 'uniqueEmail', required: true },
    password: { type: 'string', modifier: 'encrypt', require: true }
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

By embracing asynchronicity everything runs through in the same style, using the same conventions and either moving forward as intended or throwing to the `catch` and being handled cleanly.

## Summary

[Obey](https://github.com/TechnologyAdvice/obey) was developed to support an area of development that often times doesn't get enough attention, or tackles validation by assuming a datasource will handle it for you.

Our team wanted a way to centralize our data validation and modelling in one place, stop relying on after-the-fact error handling and injecting validation logic into places it shouldn't be, but more than anything we wrote it to be flexible and easy to work with.