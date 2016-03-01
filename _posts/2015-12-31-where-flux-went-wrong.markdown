---
layout: post
title: "Where Flux Went Wrong"
date: 2015-12-31T10:00:00-05:00
updated: 2015-12-31T09:00:00-05:00
author: david.zukowski
categories: development javascript reactjs flux redux
cover: /assets/images/covers/spiral-stairs.jpg
description: Regain your sanity by pulling application state out of ReactJS components.
comments: true
disqus: true
---

> Already comfortable with the history around ReactJS and Flux? Skip to [Flux and Component State](#flux-and-component-state) to jump right into the problem statement.

Ahem. When [ReactJS](https://github.com/facebook/react) first entered the development scene it attracted front-end developers across the world with its promise to introduce some semblance of sanity back into the dreaded Single Page Application. The framework, commonly referred to as the **V** in MVC, popularized the concept of componentized applications; that is, everything you see on the page is a React component. "Rid yourself of complicated controllers and unintelligible views!" it shouted from the rooftops (if you're not cool with the personification of a JavaScript framework, just imagine it's Pete Hunt), and, of course, developers rejoiced and all was right in the world.

Except it never works like that. Many a blog writer saw fit to speak out against the blasphemous JSX and comingling of markup and JavaScript; a war which still rages today, though with far less fury. The intricacies of these arguments are not vital to the point of this post, but it's important to note that React didn't receive unanimous support upon release, nor can it boast such a claim even today. What is important, however, is that it introduced a paradigm shift for front end development. Developers were no longer frought with fear over a labor necessitated by nearly all applications based on jQuery, AngularJS, or any of their kin: imperative DOM manipulation. They traded that imperative complexity for something more declarative: properties (props if you're hip) enter a React component, travel through the magical lands of the render cycle and VirtualDOM and, arriving at the end of their journey, some diffing occurs and they find themselves all grown up as part of the real DOM.

The internals are complicated, but the effect is simple: no more stressing about the DOM. Still, being only the **V** in MVC, some sort of larger structure had to be built around React in order to actually do things; you know, talk to the server, respond to events, and, most importantly, write TodoMVC’s. This is where Flux comes in, and it's also about the point where I begin to argue that React really _isn't_ just the **V** in MVC because it encourages and lends itself to a not-so-MVC approach to application architecture. So what is that approach? You've probably heard of it: [Flux](https://github.com/facebook/flux). There isn't enough time to cover the full history of Flux and all 52 of its flavors, but the gist is: components/views don't manipulate application state directly, they dispatch actions that effect changes in stores, and those changes flow back through the application from top to bottom. The result: one-way data flow.

Here's what just one small piece of this (the component) might look like:

{% highlight javascript %}
import React from 'react'
import autobind from 'autobind-decorator'
import TodoStore from 'stores/todo'
import * as TodoActions from 'actions/todo'

// Sick of TodoMVC? Me too.
class TodoList extends React.Component {
  constructor () {
    super()
    this.state = {
      todos: TodoStore.getTodos()
    }
  }

  componentDidMount () {
    TodoStore.addChangeListener(this._onTodoStoreChange)
  }

  componentDidUnmount () {
    TodoStore.removeChangeListener(this._onTodoStoreChange)
  }

  @autobind
  _onTodoStoreChange () {
    this.setState({
      todos: TodoStore.getTodos()
    })
  }

  _onToggleTodoComplete (id) {
    TodoActions.toggleTodoComplete(id)
  }

  render () {
    return (
      <ul>
        {this.state.todos.map(todo => (
          <li key={todo.id} onClick={() => this._onToggleTodoComplete(todo.id)}>
            {todo.text}
          </li>
        ))
      </ul>
    )
  }
}
{% endhighlight %}

As you can see, the todos represented in the view are not managed by the component but instead live in a store. We've managed to create a single source of truth for the todos by way of the TodoStore. Some state has been eliminated from our component, but it's not perfect.

## Flux and Component State

So what exactly is the problem with traditional Flux? Well, surprisingly, it's not the verbosity of it all. Many initial abstractions sought to reduce syntactical boilerplate but missed something so painfully obvious it hurts to look back on and realize that you didn’t see it either. _The real issue with its design is that application state (read: stuff from stores) must be applied to local component state_. How does one go about testing a component given this architecture? Well, now that it’s coupled to one or more stores, you’ll have the added work of mocking stores and actions before you can properly determine what your component looks like at the end of it all (and don't forget to check the store shape, too). Yes, the core problem with this pattern is the usage of `this.state`.

> State is the root of all evil
> - Pete Hunt

So of course React avoids state, right? Right Pete Hunt?

<div style="text-align:center;margin: 2rem auto;">
  <img src="https://media.giphy.com/media/3xz2BWKBZeM8hNWOFG/giphy.gif"/>
</div>

Proofing this post sounds like I have a beef with Pete Hunt, but I'm only kidding; I wouldn't be where I am today without his [inspirational talks](https://www.youtube.com/watch?v=KtmjkCuV-EU). However, in all seriousness, state is in fact a core feature of React components -- it's literally called `this.state` -- and it is the yin to a component's yang (props). Local state makes it difficult to determine how a component will render because the logic determining its output is internal to the instance and can change without you ever having known, and that’s just not cool, man. What you have here is a rotten case of an impure function and, if that’s not enough to set you to quaking, not only does that make testing more difficult, your component’s dependence on a specific Flux store prevents it it from being reused in different contexts.

Looking back at our previous example, what makes a `<TodoList />` component so special that it needs to know how to retrieve its own data? Its objective is simply to render a list of todos, maybe have a handler in there to toggle completion, but nothing more, and even that handler can be passed via props; it has no need for internal state. Yet all of the early flux abstractions, while often reducing the amount of boilerplate needed to apply some global state to local component state, still did exactly that: relied on `this.state`. We're not much better off than in the pre-React days at this point; yes, we've gained some benefits with the VDOM and declarative rendering, but we're still left with local state that severely complicates testing, couples components to specific stores, and increases the application's cognitive overhead (I will jump through hoops all day not say "it makes applications difficult to reason about").

We need a new approach.

## A Better Way Forward

So what can be done resolve this predicament? Enter [Redux](https://github.com/rackt/redux), a Flux paradigm that is better for all the ways that it eschews Flux's original implementation. There are many things that make Redux great, but the focus for this post is something specific that I find many Redux posts gloss over: [react-redux](https://github.com/rackt/react-redux)'s `connect` decorator. What [Dan Abramov](https://twitter.com/dan_abramov), Redux’s creator and our lord and savior, figured out was that higher-order components could be used to abstract away the store subscription in a way that not only reduced boilerplate, but totes flipped the script on us and altered how application state enters a component. Let’s take a look:

{% highlight javascript %}
import React from 'react'
import { connect } from 'react-redux'
import { actions as TodoActions } from 'modules/todo'

// Notice that we can export the raw class here as a named
// export, which means we can easily use the non-connected
// version in our tests or elsewhere in the application.
export class TodoList extends React.Component {
  static propTypes = {
    dispatch: React.PropTypes.func.isRequired,
    todos: React.PropTypes.array.isRequired,
  }

  _onToggleComplete (id) {
    this.props.dispatch(TodoActions.toggleTodoComplete(id))
  }

  render () {
    return (
      <ul>
        {this.props.todos.map(todo => (
          <li key={todo.id} onClick={() => this._onToggleTodoComplete(todo.id)}>
            {todo.text}
          </li>
        ))}
      </ul>
    )
  }
}

export default connect(state => ({
  todos: state.todos
})(TodoList)
{% endhighlight %}

Now, since the Redux [documentation](http://rackt.org/redux/index.html) is awesome I’m not going to spend time covering this in great detail, but we’ll discuss the most important point: where is the `<TodoList />` component receiving the todos from? And I don’t mean that they live in a store, but rather that they are entering the component as **props**.

The `connect` decorator is a higher-order component, a React component that wraps (i.e. renders) another component. Component state hasn't completely packed its bags, since it is still used within the component generated by `connect`, but it's been abstracted away and we don't have to fret over it. We invoke `connect` just like we would any other function, passing it some arguments (in this example we provide `mapStateToProps` to tell it exactly what state slice we want from the global state) before finally handing it our component. When the higher-order component renders, it uses the arguments we provided to determine what props to pass down. That's right, our component is rendered _inside of it_, which means we receive application state just like we would anything else in React land: as good old fashioned props.

This seemingly simple change has enormous benefits in practice:

  1. The base component (`TodoList`) can now be tested entirely independently from any stores or global state. We can simply import it into our favorite test suite, pass it some props, and see how it renders. No need to mock any stores.
  2. The base component can be freely shared across the application, since the class isn't directly tied to any one store. We can wrap it in entirely different higher order components, `connect` it with totally different state selectors, or even just pass it a plain-old array of todos.
  3. You can now rest easy knowing that if you give your component a set of props, you know _exactly_ how it will function.

The point is: the component no longer cares where it gets its data from, that's not its concern (and rightly so). And though in this case we're using `react-redux` to create the container, the component itself is not actually tied to any specific framework and now behaves just like any other simpleminded React component. There are a slew of other benefits to this pattern, such as the ability to implement performance optimizations for state selectors (see: [reselect](https://github.com/rackt/reselect)), but at the end of the day the important part is that our components are once again sane and devoid of local state.

## In Summary

State is a major contributor to application complexity, and looking back it's part of the reason why (to many, not all) React seemed like such a breath of fresh air after the two-way data binding craze. It showed us that we could break even the most complex applications down into managable pieces, so let's not turn our backs on that by reintroducing state where it's not needed. It's been argued, quite convincingly, that component state be [avoided](http://reactkungfu.com/2015/09/common-react-dot-js-mistakes-unneeded-state/), even outside of the context of Flux, and doing so even opens up [potential future optimizations](https://facebook.github.io/react/blog/2015/10/07/react-v0.14.html#stateless-functional-components). Sometimes all you need is a new way of looking at things.

Thanks Dan.
