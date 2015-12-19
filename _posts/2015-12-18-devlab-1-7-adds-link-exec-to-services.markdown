---
layout: post
title:  "DevLab 1.7 - Links and Exec on Services"
date:   2015-12-18 10:00:00
categories: devlab services
author: kent.safranski
cover: /assets/images/covers/container.jpg
description: We've been working hard on DevLab. The latest version, 1.7.0, adds the ability to use exec and links on services.
---

If you've used [DevLab](https://github.com/TechnologyAdvice/DevLab) you know a core capability is spinning up services. Services get linked into your container and make them available ephemerally for easy testing of databases, APIs, micro-services, etc.

With version 1.7.0 we've extended the capabilities of these services allowing you to link services and `exec` tasks after the services start.

## Linking

The new `link` config allows you to link multiple services. This is extremely powerful if, for instance, you want to run several micro-services and have them share a datasource.

In the `services` section of your `/devlab.yml` file this looks like:

{% highlight yaml %}
services:
  - someDatabase:
      name: foo
      persist: false
  - someApplication:
      name: bar
      persist: false
      link:
        - foo
{% endhighlight %}

Note the `link` in `someApplication` (named `bar`) references the earlier created `someDatabase` via it's name (`foo`). It's that simple. When `someApplication` loads up it will have `someDatabase` available.

Keep in mind the loading of services is done synchronously so you need to have the services in right order so they are available for the other (later) services to reference them.

## Exec

As of version 1.6.0 DevLab supports an `exec` config for any of the services. This is similar in format to the `tasks` defined at the end of your `/devlab.yml` except that this will be called immediately after the service container is started.

{% highlight yaml %}
services:
  - mongo:3.0:
      name: mongodb
      env:
        - DB_ROOT_PASSWORD=foo
      expose:
        - 27017:27017
      persist: false
      exec: |
        echo "Some task"
{% endhighlight %}

In the above snippet the `exec` task would simply run `echo "Some task"` after the container for Mongo successfully started.

---

We're continuing to come up with new ways to improve the capabilites and power of [DevLab](https://github.com/TechnologyAdvice/DevLab). Check back on this blog for updates frequently!



