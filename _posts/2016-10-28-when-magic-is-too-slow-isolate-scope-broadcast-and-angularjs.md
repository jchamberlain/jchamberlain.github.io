---
layout: post
title:  "When Magic's Too Slow: Isolate Scope, $broadcast, and AngularJS"
date:   2016-10-28
tags: [angular, javascript]
excerpt: The $broadcast beats the $digest, all around the town.
---

The other day I set about refactoring one of our AngularJS directives. Near the top of its controller I noticed the contents of a `$scope.$on()` callback suspiciously wrapped in a `$timeout()` AND a `$scope.$apply()`. There was no comment, but I know the developer who wrote this directive and he certainly wouldn't write something like that unless it solved a real problem.

<section class="callout secondary">
  <p><b>Sidenote:</b> the "Hail Mary" <code>$timeout()</code> is a code smell. If chucking your code into the future magically fixes a bug, it's almost always a sign that you either 1) don't understand how your code works, or 2) have written your code in the wrong place (e.g., in <code>controller</code> instead of <code>link</code>).</p>
</section>

## What's the problem
I removed the `$timeout()` and extra `$scope.$apply()`, and everything seemed to work. Odd. I went on with my refactoring and a few minutes later found a seemingly impossible bug: after being changed on a parent scope, models on the directive's scope were keeping their old values.

To be clear, I knew this could happen when using the `controllerAs` setting if you forget to include `bindToController: true`. But this was an older directive still using `$scope` directly without `controllerAs`, so that wasn't the problem.

I quickly tracked down the following sequence of events:

1. The directive is created with an isolate scope that includes a model called `map`. Parent and directive thus share the same JavaScript object in their scopes.
2. A parent controller changes a few things and *replaces* `map` with a new object, then sends an event via `$scope.$broadcast()` notifying of the changes.
3. The directive catches the event via the `$scope.$on()` I found above, and its callback makes a decision based on `$scope.map`.
4. Inside the callback, `$scope.map` **does not equal** the parent's `map`, and the incorrect view is rendered.
5. As a `$timeout()` demonstrates, after 20 or so milliseconds `$scope.map` **does** equal the parent's `map`. Huh?

Take a look at this [JSFiddle I made to test it out](https://jsfiddle.net/3mc5zv13/1/
). Put text in the field and hit update to see the outdated scope.

## Why??
Some of you already know what's going on, but I certainly didn't, so hear me out.

I was at first flabbergasted because I thought the parent was simply modifying properties on `map`. Cats aside, how is it possible for the same JavaScript object to have two values at the same time? But as I found in 2) above, this bug happened only when `map` had been *replaced*, which made a bit more sense.

Still, AngularJS scopes use prototypal inheritance, right? If two scopes are prototypically linked, then my flabbergastation is still justified, because they're still sharing the same JavaScript object. We've just shifted the problem one level up.

And there's the rub: **isolate scopes in AngularJS are not prototypically linked**. I don't remember reading this in the docs, but a quick Google search yielded Ben Nadel's <a href="https://www.bennadel.com/blog/2725-how-scope-broadcast-interacts-with-isolate-scopes-in-angularjs.htm" target="_blank">exploration of $broadcast and isolate scopes</a>, which says as much. Basically, normal, prototypal scopes are kept up to date by the JavaScript engine; isolate scopes are updated manually by AngularJS, **causing a delay**.

Inside our `$scope.$on()` callback, `$scope` is out of date, and there's nothing you can do about it.

## Moral of the story
Wherever possible, use event callbacks only to set–not get–models on scope. If models have been updated, they can be sent as part of the event so that listeners are explicitly given what they need.

I ended up passing `map` with the event, and this fixed the whole problem. It's left a bitter taste in my mouth realizing the promise of an up-to-date scope comes with strings attached, but the code's cleaner now nonetheless, and that's what I set out to do.