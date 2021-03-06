---
layout: post
title: "Functional Library: Null"
tags: []
---

# Functional Library: Null

Tony Hoare famously described the invention of null references as a [Billion
Dollar Mistake](http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare).

Nulls are something we need to deal with in almost any language. Any value
that can be null must be null checked. An example of a very common error that
will arise when nulls are present is:

> PHP Fatal error:  Call to a member function bar() on a non-object in foo.php
> on line n

<center>
    ![null](/img/funlib-null/null.png)
</center>

One of the ways to deal with this problem is the Null Object pattern, that
usually works quite well for behaviour, e.g. `NullLogger`. But it does not
work well at all for values, e.g. `NullAddress`.

## Option

Luckily, there is a better Option.

How often have you written this piece of code:

~~~php
$user = $repo->find($id);

if (!$user) {
    return null;
}

$address = $user->getAddress();

if (!$address) {
    return null;
}

return $address->asText();
~~~

Have you ever thought to yourself *there must be a better way*?

There is a better way.

~~~php
return $repo->find($id)
             ->map(method('getAddress'))
             ->reject(null)
             ->map(method('asText'));
~~~

All of the null checks are gone. It is now just one single expression that
describes the calls.

**Option** (aka Option Type) is a type that encodes an optional value. In
other words, you can either return something, or nothing. This is quite
similar to returning `null`. However, because everything is wrapped in an
`Option` object, you no longer need to have null checks everywhere.

<center>
    ![map option](/img/funlib-null/map-option.png)
</center>

## Some wraps a value

The previous example needs a bit of explanation. First of all,
`Repository::find()` is no longer returning `User|null`, it is now returning
`Option<User>`.

Some and None are subtypes of Option. These are the two possible types that
an Option can be.

**Some** is an object that represents (and wraps) a value. You return a Some
when you are not returning null.

**None** on the other hand represents the lack of a value. It is more or less
the equivalent of a `null`.

Here is an example of how `find` could be implemented to return an `Option`.

~~~php
use PhpOption\None;
use PhpOption\Some;

function find($id)
{
    $user = $this->em->find(User::class, $id);

    if (!$user) {
        return None::create();
    }

    return new Some($user);
}
~~~

Now it turns out that such a construction rather common. So there is a
shortcut to achieve the same thing.

~~~php
use PhpOption\Option;

function find($id)
{
    $user = $this->em->find(User::class, $id);

    return Option::fromValue($user);
}
~~~

That's how you produce an Option object.

<center>
    ![produce](/img/funlib-null/produce.png)
</center>

## Map

A common way to *consume* an option is to use `map`. If you read the previous
post on iteration you may be confused at this point. Doesn't map refer to
mapping a function over a *sequence*?

Well, it turns out that many of the things that apply for sequences can be
generalized to support other types of containers too. Yes, a sequence is just
a container for a bunch of values.

An Option is just a container for an *optional* value. Just as you can map
over a sequence to create a new sequence, you can map over an Option to create
a new Option.

A different way of thinking about it is this. Instead of calling a function on
a value:

~~~php
fn($foo);
~~~

You ask the Option container to apply a function for you:

~~~php
$foo->map('fn');
~~~

In case of **Some**, `map` takes the value out of the container, runs it
through the function that was passed in, then returns a new **Some**
containing the transformed value.

<center>
    ![map some](/img/funlib-null/map-some.png)
</center>

---

In the case of calling `map` on **None**, map will not call the provided
function at all. It will just return another **None** instead.

<center>
    ![map none](/img/funlib-null/map-none.png)
</center>

This means you can call `map` on **None** as many times as you want, it will
just ignore the calls.

## Method

Just as a little side-note, the `method` function in the original code sample
is this little helper (also present in `nikic/iter`):

~~~php
function method($name)
{
    return function ($obj) use ($name) {
        return $obj->$name();
    };
}
~~~

It creates a callable that will call the given method on any object it
receives.

## Reject

The `reject` call is a negative filter. If the Option's value matches the
rejected value, it returns **None**.

Therefore, `reject(null)` will turn `null` values into **None**. At least it's
one way of doing that conversion.

> Note: Another way of dealing with this is to make `getAddress` return a
> **Some** or **None** directly and using `flatMap` instead of `map`.

## Get

`map` is used to transform an Option.

However, that will return a new Option object. So in order to get the actual
value out, you need to use `get` or one of its variants. This means that you
can use Option in most of the code internally, and then just call `get` at the
very end.

Going back to the original example, suppose that code were the body of a
`getAddressTextForId` function. The caller of that function will get an
Option, and will have to unwrap it.

~~~php
$addressText = getAddressTextForId($id)->get();
~~~

<center>
    ![get](/img/funlib-null/get.png)
</center>

However, if this is a None, you will get a `RuntimeException` with message:
*None has no value*. In most cases, this is not a very nice way to fail.

For that reason, there are some alternatives, such as `getOrElse`, which takes
a default value to use in case of None.

~~~php
$addressText = getAddressTextForId($id)
                ->getOrElse('No address provided.');
~~~

<center>
    ![get or else](/img/funlib-null/get-or-else.png)
</center>

`getOrCall`, which takes a function that produces a value in case of None.

~~~php
$addressText = getAddressTextForId($id)
                ->getOrCall('makeDefaultAddress');
~~~

And also `getOrThrow`, which is the same as the default behaviour of `get`,
but allows you to throw a custom exception instead.

~~~php
$addressText = getAddressTextForId($id)
                ->getOrThrow(new AddressNotFoundException());
~~~

This covers the most common cases of unwrapping. There are a few more ways to
consume Option, look at the `Option` class if you're interested.

## Library

The above example is based on [Johannes
Schmitt](https://twitter.com/schmittjoh)'s impressive
[PhpOption](https://github.com/schmittjoh/php-option) library. Take a look at
the [blog post](http://jmsyst.com/blog/simplifying-algorithms-with-options) he
published yesterday. The implementation of the library is strongly inspired by
the option type available in [Scala](http://scala-lang.org).

A very common problem in programming is that of null references. We often
forget to check if a value is null. In dynamically typed languages, we have
even less chance to know if a function could return null. And if we do put in
the null checks, they add horrible clutter to the code base.

The Option type solves this problem. It wraps values in a container. It forces
callers to `map` their transformations. It allows nulls (represented as None)
to propagate without any problems.

---

<center>
    Consider it.
</center>

---

The **Option** type is the same thing as the **Maybe** monad in Haskell. If
you're interested: [Functors, Applicatives, And Monads In Pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html),
[Taking Monads to OOP PHP](http://blog.ircmaxell.com/2013/07/taking-monads-to-oop-php.html).
