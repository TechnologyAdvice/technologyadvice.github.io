---
layout: post
title: "Make React Forms Simple Again"
date: 2016-07-05 12:00:00
categories: reactjs forms javascript development
author: david.zukowski
cover: /assets/images/covers/water.jpg
description: Eliminate local state from your forms and welcome composeable higher-order components
---

Forms are fundamental to modern CRUD apps. Still, they are uninteresting for a developer and at best not profoundly unpleasant for the user. If you are a front-end application developer, chances are they are a core component of your job. Yet, despite this, [developing them is probably something you never actually enjoy doing](http://www.merriam-webster.com/dictionary/masochism). If this is the case, why do we spend so much time [rewriting the same functionality]((https://github.com/search?l=JavaScript&q=form&type=Repositories&utf8=%E2%9C%93) every time a new framework comes around? Just look at how many times we as a community have re-implemented something as simple as a dropdown just to get it to fit with the next framework or design architecture that comes along. It's a massive drain on developer resources, not to mention sanity.

I cannot offer a solution to this problem. There will never be a form framework comprehensive enough to solve all of our problems and still meet ever-changing design requirements. Instead, my approach is to limit how much an application relies on 3rd-party solutiions -- "solution" becoming more ironic as time moves on and features change. This is more difficult than it sounds, but forms are an easy place to start.

Forms are, at heart, basic I/O. They take some user input, perform validation, and submit that data to the server. From a high-level, a form is a model with setters (input), and the output of that form is a function of that model and any validation built around it. To some extent, a form library cannot be entirely generic and still practical; one designed for Angular is not well suited for React, Vue, et al. However, looking deeper, what _isn't_ wholly necessary is a Flux, Redux, Redux-Bootstrap, MobX, or any