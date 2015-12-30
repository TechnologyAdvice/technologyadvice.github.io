# TADevelops Blog

**[technologyadvice.github.io](http://technologyadvice.github.io/)

The goal of this project is to provide a platform for all members of the [TechnologyAdvice](http://www.technologyadvice.com) development team to contribute to a blog dedicated to topics surrounding work at TA.

## Contributing

To contribute to the blog please clone this repo locally. Create a branch for the blog post you would like to write with the following format:

```
---
layout: post
title:  "Title of Post"
date:   2015-11-01 10:00:00
categories: space separated labels
author: fname.lname
cover: /assets/images/covers/COVER_IMAGE.png
---

Contents of blog post...
```

A `devlab.yml` file is provided to make running the blog in a container simple. To view your post simply run `lab serve` after [setting up DevLab](#setting-up-devlab) on your machine.

When you have written your post and feel it is complete, submit a PR with basic information. Have someone else on the team review the PR and (when ready) accept it.

Once the post has been thoroughly reviewed and approved merging into `master` will publish the post on the live blog.

## Setting up DevLab

1. Install VirtualBox and Homebrew
1. Brew install Docker and Docker Machine
1. With Docker Machine create a VM called vbox-dev using the Oracle VirtualBox driver
1. Test out your setup by following along with [Containerize Your Local Dev in Minutes with DevLab]
1. With your environment configured you can run lab start to begin development (see `devlab.yml` for available tasks)

[Containerize Your Local Dev in Minutes with DevLab]: http://fluidbyte.net/docker/devlab/nodejs/2015/10/15/containerize-your-local-dev-in-minutes-with-devlab/
