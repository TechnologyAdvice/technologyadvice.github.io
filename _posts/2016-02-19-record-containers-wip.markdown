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

If you haven't been able to tell by now, I'm a pretty [huge fan of React, Redux, and higher-order components](). For this example I'll be using Redux, but the concept is largely applicable to any similar Flux or Flux-like implementation, namely those that make use of higher-order components.

{% highlight javascript %}
{
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
                    friends: [2, 3], // an array of userId's
                },
            },
        },
    },
}
{% endhighlight %}

{% highlight javascript %}
type Props = {
    id: string,
    fetchRecord: Function,
    state: {
        record: Object,
    },
};
const DEFAULT_RECORD_APPLY = (record) => ({ record })
const RecordContainer = (applyRecordToProps = DEFAULT_RECORD_APPLY) => {
    return (...connectArgs) => (WrappedComponent, PlaceholderComponent) => {
        class ConnectedRecordContainer extends React.Component<void, Props, void> {
            componentWillMount() {
                if (this.props.id) {
                    this.props.fetchRecord(this.props.id)
                }
            }

            componentWillReceiveProps(nextProps) {
                if (nextProps.id && this.props.id !== nextProps.id) {
                    this.props.fetchRecord(nextProps.id)
                }
            }

            // if you're using immutable data (and can safely do this given the structure of your
            // application), you can get a free win here.
            shouldComponentUpdate(nextProps) {
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
const UserRecordContainer = RecordContainer((state, ownProps) => ({
    state: state.users.byId[ownProps.id],
}), { fetch: someAjaxServiceToFetchUserById })

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