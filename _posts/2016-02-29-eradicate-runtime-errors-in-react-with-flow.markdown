---
layout: post
title: "Eradicate Runtime Errors in React with Flow"
date: 2016-02-28 18:00:00
categories: reactjs flow javascript development
author: david.zukowski
cover: /assets/images/covers/water.jpg
description: Catch errors before they happen by statically analyzing your code with Flow.
comments: true
disqus: true
---

[Flow](http://flowtype.org/) is pretty awesome.

Oh, more details... Ok, so, Flow is great for large or growing codebases that could use a hand in the [static analysis](https://en.wikipedia.org/wiki/Static_program_analysis) department. Even outside of the context of React ([heh](https://facebook.github.io/react/docs/context.html)), Flow shines by ensuring type safety across modules; that is, it allows you to annotate variables, functions, et al. with type annotations that Flow can parse and use to spot errors in your application before they happen.

Why Flow? Our team settled on Flow because we already had multiple fairly substantial React codebases with complicated-enough build systems that we wanted to avoid modifying (the build systems), which pretty much discounted TypeScript, at least for the immediate gains we were seeking. Luckily, Flow gave us the opportunity to incrementally add types to our applications with a simple `/* @flow */` pragma and little else.

If you haven't given it a chance yet I strongly advise [giving it five minutes](https://signalvnoise.com/posts/3124-give-it-five-minutes). In fact, despite balking at some of the syntax at first, the first time I saw a `Property cannot be accessed on possibly null or undefined value` on a [Maybe Type](http://flowtype.org/docs/nullable-types.html#_) I pretty much spammed our team's Slack channel in excitement.

It was, as the thesis states, awesome.

## Type Safety and React

Unlike what some [recent rumors](https://www.reddit.com/r/reactjs/comments/44yqge/fyi_proptypes_are_on_their_way_out/czubij1) may have implied, `PropTypes` are not going anywhere, at least any time soon. However, that doesn't mean they're perfect; there are a number of places where they fall short, notably:

* `PropTypes` are not statically analyzable. When you encounter an error it's during development and only once the problem component has rendered. That's far too late, and the fact that it's caught at runtime means you're likely to miss errors that could end up biting you when you roll out to production.

* They add unneeded `noop` bytes in production. Yes, not the biggest of deals, and it [is possible to remove them](https://github.com/oliviertassinari/babel-plugin-transform-react-remove-prop-types), but that's Yet Another Plugin for your growing webpack + babel + rosetta stone build pipeline.

* As the name implies, the type checking is limited to component properties: they cannot be used to type check any other parts of your codebase.

* Finally, `PropTypes` are simply less powerful. They can, to a degree, ensure that the props entering a component match your expectations, but that's where their utility ends; they can make no guarantees that you, for example, do not try to access non-existent properties on objects passed through props, nor do `PropTypes` check _how_ those props are used throughout a component.

Flow solves all of these concerns and is an invaluable addition to the typical workflow, perhaps as a [pre-commit hook](http://technologyadvice.github.io/the-power-of-the-precommit/) or as part of your CI process. But enough with the words already, let's write some code. Everyone is rightfully sick of [TodoMVC](http://todomvc.com/), so we'll change it up by doing some TodoMVC _with Flow_.

{% highlight javascript %}
/* @flow */
import React from 'react'
import autobind from 'autobind-decorator'
import TodoListItem from './TodoListItem'

type Props = {
  todos: Array<Object>,
}
class TodoList extends React.Component {
  props: Props;

  // Yes, I could use an arrow function, I get it.
  @autobind
  _handleCompleteTodo(id: string): void {
    // do something to mark this todo as complete
  }

  render(): React.Element {
    return (
      <ul>
        {this.props.todos.map((todo) => (
          <li key={todo.id}>
            <TodoListItem
              todo={todo}
              onComplete={() => this._handleCompleteTodo(todo.id)}
            />
          </li>
        ))}
      </ul>
    )
  }
}
{% endhighlight %}

* **Note 0.14.0**: As of [Flow v0.22.0](https://github.com/facebook/flow/releases/tag/v0.22.0), `ReactElement` and its kin are no longer implicitly available on the global scope. If you are, for example, [creating a declaration](http://flowtype.org/docs/declarations.html#_), you will have to import React and reference the types off of the `React` namespace, e.g. `ReactElement` -> `React.Element`.
* **Note 15.0.0** `void` means "no return value" aka `undefined`.

Ok, back to it. So this is pretty cool, we've added some fairly basic type annotations to our `<TodoList />` component, where we defined the property `todos` as an array of objects and annotated the component's two methods. I'm not going to cover the fact that Flow will type check these properties, as that's Flow's fundamental offering, so let's continue by taking a look at `<TodoListItem />`:

{% highlight javascript %}
/* @flow */
import React from 'react'

type Props = {
  todo: Object,
  onComplete: Function,
}
class TodoListItem extends React.Component {
  props: Props;

  render() {
    return (
      <div>
        <h3>{this.props.todo.text}</h3>
        <button onClick={this.props.onComplete}>Complete</button>
      </div>
    )
  }
}
{% endhighlight %}

Not bad, our type of `todo` (`Object`) fits right in with what `<TodoList />` expects for `todos` (an array of `Objects`), but we can take this a step further. As many of you will recognize, this looks very similar to `PropTypes.shape`:

{% highlight javascript %}
// ...
type Props = {
  todo: {
    id: string,
    text: string,
    completed: boolean,
  },
  onComplete: Function,
}
// ...
{% endhighlight %}

We've now defined the shape of a `todo` inline, however, we're not done yet. We can improve this by creating (and exporting!) a reusable [Type Alias](http://flowtype.org/docs/type-aliases.html#_):

{% highlight javascript %}
// ...
export type Todo = {
  id: string,
  text: string,
  completed: boolean,
}

type Props = {
  todo: Todo,
  onComplete: Function,
}
// ...
{% endhighlight %}

This type alias can then be imported from other components or even made into an app-wide declaration if you find yourself using it often enough. So, how does Flow help us? Well, let's tweak our `<TodoListItem />`'s render method to display a bit more information:

{% highlight javascript %}
// Our updated Todo:
export type Todo = {
  id: string,
  text: string,
  completed: boolean,

  // the `?` signifies that this property might not exist.
  assignedUser?: {
    id: string,
    name: string,
  },
}

// and the new and improved render method:
render() {
  const { todo } = this.props

  return (
    <div>
      <h3>{todo.text}</h3>
      <h4>Assigned to: {todo.assignedUser.name}</h4>
      <button onClick={this.props.onComplete}>Complete</button>
    </div>
  )
}
{% endhighlight %}

If we run `flow check` on this, we get an error:

<div style="text-align:center;margin: 1rem auto;">
  <img src="/assets/images/posts/eradicate-runtime-errors-in-react-with-flow/flow-maybe-type-error.png" alt="Flow can spot accessors on nullable types" title="Flow can spot accessors on nullable types" />
</div>

Flow recognized that `assignedUser` might not exist and correctly fails the analysis. This sort of safeguard is invaluable as applications grow since it's no longer left to the developer to account for these scenarios; even the best of us can forget these things.

Here's how a fixed version of this component might look:

{% highlight javascript %}
// ...
class TodoListItem extends React.Component {
  props: Props;

  render() {
    const { todo } = this.props

    // a bit roundabout, but y'know, clarity...
    let assignedToContent
    if (todo.assignedUser) {
      assignedToContent = `Assigned to: ${todo.assignedUser.name}`
    } else {
      assignedToContent = 'This is not assigned to anyone!'
    }

    return (
      <div>
        <h3>{todo.text}</h3>
        <h4>{assignedToContent}</h4>
        <button onClick={this.props.onComplete}>Complete</button>
      </div>
    )
  }
}
{% endhighlight %}

### Function Properties

Now let's say you are, rightfully, a fan of the React ESLint rule [`jsx-no-bind`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md). In this case you will probably convert your `<TodoList />` component to look something more like this:

{% highlight javascript %}
// ...
import type { Todo } from './TodoListItem'

type Props = {
  todos: Array<Todo>,
}
class TodoList extends React.Component {
  props: Props;

  @autobind
  _handleCompleteTodo(id: string): void {
    // do whatever it was we were doing before
  }

  render(): React.Element {
    return (
      <ul>
        {this.props.todos.map((todo) => (
          <li key={todo.id}>
            <TodoListItem
              todo={todo}
              onComplete={this._handleCompleteTodo} // <-- important
            />
          </li>
        ))}
      </ul>
    )
  }
}
{% endhighlight %}

Notice that we are no longer creating a function with each iteration of `this.props.todos`. Whether or not we do this with `.bind` or an arrow function is beside the point; the fact is, we are now delegating this responsibility to `<TodoListItem />`. Let's dive in and see what that the modified component would look like (removing some demo code for succinctness):

{% highlight javascript %}
// ...
type Props = {
  todo: Todo,
  onComplete: Function,
}
class TodoListItem extends React.Component {
  props: Props;

  // Ok, so when a user clicks the button to complete this
  // todo, we invoke the `onComplete` callback provided by
  // our parent component, passing it our todo so it knows
  // which todo was completed.
  @autobind
  _handleComplete(): void {
    this.props.onComplete(this.props.todo)
  }

  render(): React.Element {
    return (
      <div>
        <button onClick={this._handleComplete}>Complete</button>
      </div>
    )
  }
}
{% endhighlight %}

Looks great, except there's a bug, which is... not so great. The `<TodoList />` component's `_handleCompleteTodo` method is still expecting an argument `id` of type `string`, not the entire todo object. What we have right now is still fairly equivalent to what is possible with regular `PropTypes`, meaning `Function` does not say anything about its signature or return type. Luckily, we can go a step further with Flow.

Since we know exactly how our component plans to call `onComplete`, we can explicitly define that signature in the `Props` type definition, which will require any implementers to abide by that signature. Check it out:

{% highlight javascript %}
type Props = {
  todo: Todo,
  onComplete: Function,
}

// "Let's kick it up a notch!" - Emeril Lagasse

type Props = {
  todo: Todo,
  onComplete: (todo: Todo) => any,
}
{% endhighlight %}

At this point, we've defined the property `onComplete` to be a function that takes a `Todo` as its first and only argument. We don't actually care what that function returns, it's simply a callback, and those guys are super forgettable, so we specify `any` as the return type. With this in place, let's run `flow check` and see what we get:

<div style="text-align:center;margin: 1rem auto;">
  <img src="/assets/images/posts/eradicate-runtime-errors-in-react-with-flow/flow-function-signature-error.png" alt="Flow can spot errors in function signatures across modules" title="Flow can spot errors in function signatures across modules" />
</div>

If you're incredibly attentive you may notice that this error was spotted before `onComplete` was ever even used in the child component, it was caught as the `onComplete` prop was provided to `<TodoListItem />` from `<TodoList />`.

## Pairing with Flux

Our most recent applications are built on top of the popular [Redux](https://github.com/reactjs/redux) library, which Flow has complemented incredibly nicely, though it would presumably do the same with any Flux library, or even any non-Flux architecture... well, you get the point. We'll take a brief look at two areas in particular that we've seen quick and easy benefits with:

### Action Creators

One easy place to start adding Flow type annotations to is your action creators. If you're familiar with [Flux Standard Action](https://github.com/acdlite/flux-standard-action), you could create a global declaration for this interface:

{% highlight javascript %}
type Action = {
  type: string,
  payload?: any,
  error?: boolean,
  meta?: any,
}
{% endhighlight %}

Granted, FSA is actually fairly weak in its type constraints, so you could enforce this more strictly as it suits you and your team. Still, this helps when creating your Flux modules:

{% highlight javascript %}
export const completeTodo = (id: string): Action => ({
  type: TODO_COMPLETE, // constant declared above
  payload: {
    id,
  },
})
{% endhighlight %}

If you were to return an object that didn't adhere to the `Action` type Flow would learn about it, hunt you down, and <s>kill you</s> fail your CI build.

### State Shapes

The obvious benefit here is that we can create a type alias for each reducer's shape, allowing Flow to analyze that each action handler in a reducer produces a conforming state shape. This is fairly trivial once you've annotated a function or two, since reducers are _just plain functions_, but there's an even bigger benefit to be had.

Say we have a reducer responsible for some search state, perhaps a form with a handful of fields, nothing crazy. When the filters are submitted (in this example), we update the store to cache that state for returning visitors to provide some decent UX. This is super high-level, so please forgive me for not going through it in its entirety.

{% highlight javascript %}
// your filter state shape (presumably in a redux module)
export type MySearchFilterState = {
  selectedUserIds: Array<string>,
  searchFromDate: number,
  searchToDate: number,
  hasSomeFilterEnabled: boolean,
}

// consumer component (parent smart component):
import type { MySearchFilterState } from 'my/redux/module'

class MySmartViewOrSomething extends React.Component {
  _handleSubmitFilters(filters: MySearchFilterState): void {
    // dispatch an action to cache these filters
  }

  render(): React.Element {
    return (
      // a bunch of stuff...
      <MyFormComponent onSubmit={this._handleSubmitFilters} />
    )
  }
}

// consumer component's child
type Props = {
  filters: MySearchFilterState,
  onSubmit: Function,
}
class MyFormComponent extends React.Component {
  // ...
  _handleSubmitFilters(): void {
    this.props.onSubmit({
      selectedUserIds: [...],

      // "The Witch" is pretty good, you should see it.
      someBadProperty: 'Black Phillip',
    })
  }
  // ...
}
{% endhighlight %}

The amazing part about this setup is that if `<MyFormComponent />` decides to not abide by the shape of the filters (as it fails to do above), Flow will catch it without ever having to run the application. It will not only validate the shape of those filters, but will follow them through their entire lifecycle, far outside of the reach of component properties. Whether the error is an additional/malformed property, a missing required property, or an incorrect type, Flow has you covered. If you ever change the shape of your state, Flow will help you confirm that the rest of the application continues humming along.

Flow isn't perfect, it doesn't catch everything just yet and sometimes requires you to write your code in a way that's a bit more explicit, but it's a worthy tradeoff, especially considering the fact that it's actively maintained and constantly improving. It can make guarantees about your codebase that `PropTypes` would only be able to offer as uncaught exceptions in production.

Seriously, Flow is awesome.
