---
layout: post
title:  "The Power of Precommits"
date:   2015-11-15 10:00:00
categories: git development
author: kent.safranski
cover: /assets/images/covers/checklist.png
---

When you're writing code there's typically a lot more than just putting code into an IDE. Your writing & running tests, doing static analysis, checking deploys and so on. Anywhere that automation can (intelligently) be added into your workflow can be a huge benefit. Precommits offer a way to automate repetitive task easily.

There are a number of [Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) but precommit is one of the more useful. From the previous link:

> The pre-commit hook is run first, before you even type in a commit message. It’s used to inspect the snapshot that’s about to be committed, to see if you’ve forgotten something, to make sure tests run, or to examine whatever you need to inspect in the code.

Basically, before you commit your code it runs your tasks and either a) exits `0` or b) exits with an error from whatever task was run. This means that you can do some work, get everything ready to commit and then have a final check to ensure you didn't miss anything.

Pre-commit hooks can be added in the Git config, but this only applies to your local environment. One tool we've found extremely useful at [TA](http://www.technologyadvice.com) is the [NPM precommit-hook module](https://www.npmjs.com/package/precommit-hook).

Setup is simple; install the module, makes sure you have npm scripts defined and tell it what tasks to run precommit.

#### Installation

```
npm install precommit-hook --save-dev
```

#### Add Some Scripts

In your `package.json` either use the tasks you have defined, or define some to run:

{% highlight javascript %}
"scripts": {
  "lint": "eslint /src --fix",
  "test": "mocha /src --recursive"
}
{% endhighlight %}

#### Define the Precommit

Also in your `package.json`, define the following to specify the tasks to run:

{% highlight javascript %}
"pre-commit": [ "lint", "test" ]
{% endhighlight %}

That's it! Now when anyone commits on the repo your precommit tasks will run and let them know if everything's good or if a test or lint check is failing!
