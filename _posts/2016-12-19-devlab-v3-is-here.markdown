---
layout: post
title: "Devlab v3 Is Here"
date: 2016-12-19 10:00:00
categories: docker devlab development
author: kent.safranski
cover: /assets/images/covers/container-load.jpg
description: After a year of dogfooding, a complete rebuild, and 110+ commits, we are excited to release Devlab v3.
comments: true
---

After a year of dogfooding, a complete rebuild, and 110+ commits, we are excited to release Devlab v3.

## New to Devlab?

Devlab is a CLI tool that allows you to easily spin up [Docker](https://www.docker.com/) containers with all the services, environment variables, and settings you need to run a project in a cleanroom.

If you want to run an application you're developing that requires a database (or any other services), you define the service(s), the configuration, and tasks you routinely run. Once your project is configured you run `tasks` and every time, Devlab spins up a parent container and links in any service containers you need.

For more information, [check out the Devlab repo on Github](https://github.com/TechnologyAdvice/DevLab).

## What's New In v3?

**To sum it up in 3 words: performance, performance, performance.**

Loading up and shutting down containers takes time. Not a lot, but it adds up if you're running multiple services, or services which are heavy/slow (see: MySQL).

#### Parallel Startups

This is where things started: instead of loading each service and checking for 0-code on start we load all services in parallel and only exit if something goes wrong. This means that initial startup time is only limited to your slowest service.

#### Detached Shutdown

So you run a number of services, execute your tests, and everything's golden. With previous versions you would then have to wait for the containers to exit. With v3 we've detached this process, so after your task runs, the process fires off a service shutdown script and immediately exits.

#### Unified Configuration

Previously, service configurations were parsed separately from your main project configuration, and had different keys in the devlab.yml. But in v3, the same code processes both of these things. Now, services and the primary containers are configured using the exact same options.

This also means the configuration got a LOT easier to work with. With services for instance, pre-v3:

```yaml
from: node:latest
services:
  - mongo:latest:
      name: mongodb
      expose:
        - 27017:27017
```

Now, the name of the key and the image is specified using `from`, just like in the primary service:

```yaml
from: node:latest
services:
  - mongodb:
      from: mongo:latest
      expose:
       - 27017:27017
```

#### Multiple Task Execution

In previous versions, the only way to specify multiple tasks was using a (not so well-thought-out) alias to another task. You can dig the old source-code if you're really interested in what that looks like. With v3 you simply add it to the command:

```
devlab lint test cover
```

#### Cleaner Output

Previously, output (task startup, execution, shutdown notifications) were verbose. Probably more verbose than anyone liked, so we pared them down to some basic indicators of activity and let you focus on the output of the task.

![demo](/assets/images/posts/devlab-v3-is-here/demo.gif)

#### 58% Code Reduction

Thanks in large part to the aforementioned unification of configuration, we were able to reduce the code down by a whopping 58% without cutting any of the core functionality.

This reduction also means more performant code in general.

We hope this also encourages anyone who sees an issue or area of improvement to easily jump in and drop a PR.

## Lots of Work, Lots of Improvements

We've put a lot of time into Devlab. Everyone on our team works with it, and we plan to continue developing enhancements and improvements. The latest v3 release is our biggest push on the project since its initial release and we would love your feedback!





