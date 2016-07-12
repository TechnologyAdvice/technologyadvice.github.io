---
layout: post
title: "App Analytics with Redux"
date: 2016-03-24 11:25:00
categories: analytics redux segment data monitoring
author: josh.habdas
cover: /assets/images/covers/eagle-grip.jpg
description: Learn how to track app analytics with Redux the easy way using Segment.
comments: true
disqus: true
---

So you figured out [Where Flux Went Wrong] and are shipping your app with Redux. How will you measure usage? Will you know how users are using it once it's launched? What about user authentication? You'll definitely want to track that. But how will you do it?

Will you add Google Analytics integration? Perhaps you're planning to rock your own dashboards with [Keen.io]. But what about heat maps, page read statistics, e-commerce integration, or the ability to watch video recordings of your users? Which tools will you choose? How many integrations will you need? How well will those integrations work with Redux?

If you're asking these questions it's a good thing. But the analytics tooling space is vast. And with so many options, which will you choose?

I'm going to let you in on a little tip: don't choose any of them, choose as many as you can and start experimenting.

> In this post you will learn how to enhance your reducers to connect your app to hundreds of analytics tooling integrations with a single dependency, at little to no cost and with very minimal effort.

## Analytics is about learning

Have you ever heard of Segment? If not, let me give you the skinny.

[Segment] is an analytics aggregation service providing [hundreds of analytics tooling integrations](https://segment.com/integrations) with the toggle of a switch. It's reliable and easy to work with. So easy, in fact, I [added it to my blog](https://github.com/jhabdas/habd.as#features). Their pricing model is moderate, even at scale. And, best of all, you can try out dozens of popular integrations such as Google Analytics for free, without any development experience needed.

Redux users integrating with Segment are in luck! Turns out the team over at [Rangle] have graciously open sourced a slick middleware library called [`segment-redux`](https://github.com/rangle/redux-segment), making Redux integration with Segment so easy you could do it with a hand tied behind your back. Huge props to [@bertrandk](https://github.com/bertrandk) on his efforts maintaining it.

Let's install the middleware and start tracking, shall we?

## Installing the Redux middleware

Unless you're using a space-age abstraction like [Redux Providers & Replicators], or aren't even using React at all, you're probably using something like React Router alongside [`react-router-redux`](https://github.com/reactjs/react-router-redux) to handle routing in your application. If so, you'll be tickled to know that, once the middleware is installed, you'll get application page view reporting out of the box.

**Install the middleware package** with `npm i -S redux-segment` and follow along with the [two-step install instructions](https://github.com/rangle/redux-segment#installation) to connect it with Redux.

The middleware installation instructions currently assume you're building a fat client JS app and will be using [`Analytics.js`](https://github.com/segmentio/analytics.js). But you could just as easily swap in [`analytics-node`](https://github.com/segmentio/analytics-node) if you're building a Node app instead. Using `redux-thunk` for async data flow? No problem, the middleware supports that too.

## Connecting with Segment

If you've already signed up with an account and **created a _Project_ for your app** in Segment all you need to do is `_.plop` the `WRITE_KEY` provided into your app while initializing, and let the Redux middleware do the rest. The key is safe to share, so you don't need to worry about [locking it up](http://technologyadvice.github.io/lock-up-your-customer-accounts-give-away-the-key/).

## Sending an event

Once you've installed the `redux-segment` middleware and connected with Segment you can start tracking events immediately. As mentioned earlier, users of React Router Redux will enjoy out of the box support and will see _Page_ events start flying the moment they nagivate between routes. Additional events can be set-up with just a few lines of code in your reducer.

Here's an example _Identify_ event we use to track user login events on some of our React apps here at TA:

```javascript
import { EventTypes } from 'redux-segment'

export const loginSuccess = (token, user) => ({
  type: SESSION_LOGIN_SUCCESS,
  payload: {
    token,
    user,
  },
  meta: {
    analytics: {
      eventType: EventTypes.identify,
      eventPayload: {
        userId: user.id,
        traits: (() => {
          const { email, firstName, imageUrl, lastName, phoneNumber } = user
          return {
            avatar: imageUrl,
            email,
            firstName,
            lastName,
            phone: phoneNumber,
          }
        })(),
      },
    },
  },
})
```

Notice how we've taken a simple reducer and added a `meta` property called `analytics`, along with an event type and payload. That's all there is to it! See the middleware [Usage section](https://github.com/rangle/redux-segment#usage) for different event types, supported routers and additional documentation.

## Track to your heart's content

Now that you have an arsenal of analytics tooling integrations at your fingertips you can ask your stakeholders what they want to measure, instead of asking them how they want to go about measuring it.

I've used Segment in the past to set-up and scale several single-page apps tracked by Google Analytics, track application errors with Sentry and watch users navigate a site to discover UX issues. Currently I'm working on an AWS client-side monitoring solution leveraging [Lambda] and webhooks. And I get geeked every time someone shows me something new, so please don't hesitate to gush about your favorite analytics tools in the comments section below.

[Keen.io]: https://keen.io/
[Lambda]: https://aws.amazon.com/lambda/
[Rangle]: http://blog.rangle.io/
[Redux Providers & Replicators]: https://medium.com/@timbur/react-automatic-redux-providers-and-replicators-c4e35a39f1
[Segment]: https://segment.com/
[Where Flux Went Wrong]: http://technologyadvice.github.io/where-flux-went-wrong/
