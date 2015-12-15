---
layout: post
title:  "Weak Tests Hurt"
date:   2015-11-15 10:00:00
categories: git development
author: kent.safranski
cover: /assets/images/covers/failed-test.jpg
description: Writing tests isn't exactly an art but there are some really common mistakes made when writing them that can cost you big in the end.
---

Hopefully you're writing tests for your code, if not you really should. I don't look at testing as an exact science; it's something that needs to be done, but sometimes the happy-path (i.e. having a pasing suite of tests) becomes the goal instead of focusing on what the tests _actually test_.

> I should be able to _ever so slightly_ alter a test and make it fail.

If the above statement isn't true you should probably reevaluate your tests. Tests should ultimately be brittle. They, unlike your code, should perform one, single task and exit; pass/fail.

A good test is one that can fail easier than it can pass.

### An Example

Let's say you're retrieving data from a database. Pretty standard CRUD stuff. For our example we'll retrieve a list of users:

{% highlight javascript %}
[{
	"name": "John Smith",
	"email": "jsmith@email.com"
}, {
	"name": "Bill Jones",
	"email": "bjones@email.com"
}, {
	"name": "Sarah Williams",
	"email": "swilliams@email.com"
}]
{% endhighlight %}

Now, let's pretend you have an interface (object) for interacting with this data source and model, just core CRUD operations, something like:

{% highlight javascript %}
const user = new SomeCRUDUtil();
{% endhighlight %}

### Weak Assumptions

You don't have to look to hard to find where a weak assumpation could be made. Creating records is pretty straight-forward, but what about retrieval?

If you query the entire collection you expect to get an array back, right?

{% highlight javascript %}
user.read().then(data => expect(data).to.be.an.array)  
{% endhighlight %}

Yes, if everything works right you get back an array, but it's not fragile. Assume `read` takes an object-query parameter and I plug in `{ name: 'Bob Smith' }`.

Does that return an empty array? Probably. There's no `Bob Smith` in my data, but an empty array is an array nonetheless.

So, improvements? Of course! Let's look at just the assertion:

{% highlight javascript %}
// Weak...
expect(data).to.be.an.array

// Better
expect(data).to.not.be.empty

// Best (assuming you seed your data like a good dev)
expect(data).to.deep.include({ 
  name: "John Smith",
  email: "jsmith@email.com"
})
{% endhighlight %}

That last assertion is not just a surface-level test, it's looking for a spot-on match.

In my personal opinion the vast number of assertions in libraries fall short. It's always best to look for an exact match, `equals` and `deep.equals` are your best friends in testing because they don't ever settle for "close enough".

Weak tests hurt your code. They hurt your code because the assumptions are an easy match which is dangerous when relying on components, modules, etc.

If my example CRUD Util returned `null` currently if nothing was matched just checking for an array might suffice. But, should that util change to returning an empty object and my dependencies update how would I know that my test was failing by simply checking the type?

---

The above is a very simple example of making weak assertions but it's very obvious. The point is not that type-checking or weak assertions are to not be used, however, they should not be relied upon.

When you're testing you code you want to be as specific as possible.

