---
layout: post
title: "Record Containers WIP"
date: 2016-02-19T10:00:00-05:00
updated: 2016-02-19T09:00:00-05:00
comments: true
author: david.zukowski
categories: development javascript reactjs flux redux
cover: /assets/images/covers/spiral-stairs.jpg
description: Stuff.
---

# Ignoring Asynchronicity with Higher-Order Components

If you haven't been able to tell by now, I'm a pretty [huge fan of React, Redux, and higher-order components](). For this example I'll be using Redux, but the concept is largely applicable to any similar Flux or Flux-like implementation, namely those that make use of higher-order components. First, though, a brief description of the problem this pattern aims to solve.

Say you have an application with a variety of widgets that are reused across multiple, disparate views (dashboards, detail pages, et. al.) and each widget is responsible for displaying information about multiple related models. Taking our favorite TodoMVC application as an example, our `<TodoListItem />` would be responsible for more than just displaying the text of that todo; instead, it may have to display information about who created that todo and what todos are dependent on it.

Disclaimer: this is an exceptionally contrived example, it is purely for (relative) ease of demonstration. Regardless of _how_ you go about fetching the data, this is likely how your store state would progress assuming you progressively load each set of data:

#### Stage 1) User Detail Page Loads
{% highlight javascript %}
{
  todos: {
    byId: {},
  },
  users: {
    byId: {},
  },
}
{% endhighlight %}

#### Stage 2) User is Fetching -> Fetched
{% highlight javascript %}
{
  todos: {
    byId: {},
  },
  users: {
    byId: {
      1: {
        error: null,
        isFetching: true,
        receivedAt: null,
        record: null,
      }
    },
  },
}

// later...

{
  todos: {
    byId: {},
  },
  users: {
    byId: {
      1: {
        error: null,
        isFetching: false,
        receivedAt: 1455849489711,
        record: {
          id: 1,
          text: 'Call Dwight',
          createdByUser: 1,
          followupTodos: [2],
        },
      },
    },
  },
}
{% endhighlight %}

#### Stage 3) Fetch Todos for User
{% highlight javascript %}
{
  todos: {
    byId: {
      1: {
        error: null,
        isFetching: true,
        receivedAt: null,
        record: null,
      },
    },
  },
  users: {
    byId: {
      1: {
        error: null,
        isFetching: false,
        receivedAt: 1455849489711,
        record: {
          id: 1,
          text: 'Call Dwight',
          createdByUser: 1,
          followupTodos: [2],
        },
      }
    },
  },
}

// later...

{
  todos: {
    byId: {
      1: {
        isFetching: false,
        receivedAt: 1455849489811,
        record: {
          id: 1,
          text: 'Call Dwight',
          createdByUser: 1,
          followupTodos: [2],
        },
      },
    },
  },
  users: {
    byId: {
      1: {
        isFetching: false,
        receivedAt: 1455849489711,
        record: {
          id: 1,
          firstName: 'Michael',
          lastName: 'Scott',
          summary: 'The best boss in the world.',
          todos: [1],
        },
      },
    },
  },
}

{% endhighlight %}

Then continue this process to retrieve the followup todos for each of the user's todos. There are a few ways that this could be achieved. It could be a responsibility of the view that renders this list of todos; when the view mounts, kick off requests that dispatch the requisite actions to insert the records into the store. But what happens when we want to use this component in multiple views? We could define data needs in our component and pass them up through the component tree, where the root node (the view/route) combines them and makes the requests in bulk.

{% highlight javascript %}
type Props = {
  id: string,
  fetchRecord: Function,
  state: {
    record: Object,
  },
};
const DEFAULT_APPLY_RECORD_TO_PROPS = (record) => ({ record })
const RecordContainer = (applyRecordToProps = DEFAULT_APPLY_RECORD_TO_PROPS) => {
  return (...connectArgs) => (WrappedComponent, PlaceholderComponent) => {

    // Create the higher-order component class
    class ConnectedRecordContainer extends React.Component<void, Props, void> {
      componentWillMount() {
        if (this.props.id) {
          this.props.fetchRecord(this.props.id)
        }
      }

      componentWillReceiveProps (nextProps) {
        if (nextProps.id && this.props.id !== nextProps.id) {
          this.props.fetchRecord(nextProps.id)
        }
      }

      // if you're using immutable data (and can safely do this given the structure of your
      // application), you can get a free win here.
      shouldComponentUpdate (nextProps) {
        return !shallowEqual(this.props, nextProps) // https://github.com/dashed/shallowequal
      }

      render() {
        const { id, state, fetchRecord, ...rest } = this.props
        const record = state && state.record

        if (record) {
          return <WrappedComponent state={state} {...applyRecordToProps(record)} {...rest} />
        } else if (PlaceholderComponent) {
          return <PlaceholderComponent state={state} {...rest} />
        } else {
          return <div />
        }
      }
    }

    return connect(...connectArgs)(ConnectedRecordContainer)
  }
}

export default RecordContainer
{% endhighlight %}

This pattern could be implemented in the following way:

{% highlight javascript %}
const applyRecordToProps = (record) => ({ user: record })

// Here we abide by the contract specified by `RecordContainer`, where it expects
// to find a property "state" that can be used to determine whether or not our
// target resource exists in the store or not.
const mapStateToProps = (state) => ({ state: state.users.byId[ownProps.id] })

// Here we tell the RecordContainer how to retrieve the target record from the store
// as well as how to fetch it. `applyRecordToProps` is for convenience, allowing us
// to name the property that the record is applied to our component as. If we don't
// specify `applyRecordToProps`, it will simply be named `record`.
const UserRecordContainer = RecordContainer(applyRecordToProps)(
  mapStateToProps, { fetchRecord: someAjaxServiceToFetchUserById }
)

// later, in another file:

const UserSummary = UserRecordContainer(({ user, ...props }) => (
  <div {...props}>
    <h3>{user.firstName} {user.lastName}</h3>
    <p>{user.summary}</p>
  </div>
))
{% endhighlight %}

With this setup, a component implementing `<UserSummary />` only has to provide it an `id` and it will do the rest. Once the user has been fetched, the component will render the "ready state" component wrapped by `UserRecordContainer`. You no longer have to worry about calling actions to fetch it yourself,

Imagine you now want to show the names of this user's friends. You _could_ assume that you retrieved that data when you fetched this user, but that might not always be true, given the scope of your application and how you've decided to consume the API.

{% highlight javascript %}
const UserFullName = UserRecordContainer(({ user }) => (
  <span>{user.firstName} {user.lastName}</span>
), ({ state }) => (
  state.error ? <span>Error loading this user</span> : <Spinner />
))

// ...

const UserSummary = UserRecordContainer(({ user, ...props }) => (
  <div {...props}>
    <h3>{user.firstName} {user.lastName}</h3>
    <p>{user.summary}</p>
    <h4>Friends</h4>
    <ul>
      {user.friends.map((userId) => (
        <li key={userId}>
          <UserFullName id={userId} />
        </li>
      ))}
    </ul>
  </div>
))
{% endhighlight %}