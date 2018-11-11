---
layout: post
title:  "Python iterables and iterators"
date:   2018-11-11 23:30:00 +0100
---

Python's `for` loop is syntactic sugar. The following code:
```py
for <var> in <obj>:
    <code>
```
is equivalent to:
```py
i = iter(<obj>)
while True:
    try:
        <var> = next(i)
    except StopIteration:
        break
    <code>
```
This even correctly handles `break`, `continue` and `else`.

In the above code what the built-ins `iter(foo)` and `next(bar)` do is (essentially) call the `foo.__iter__()` and `bar.__next__()` dunder methods and return their results.

*[dunder methods]: "double-underscore methods", also called "magic methods"

An _iterable_ in Python is, unsurprisingly, an object on which one can iterate. By that it is meant that it's an object that can be used in a `for` loop by being put in place of the `<obj>` in the code above. Thus all that an iterable is required to implement is a `__iter__()` method. This method must return something called an _iterator_. Looking again at the code above, all an iterator must do is expose a `__next__()` method which either returns the next element to iterate on or raises `StopIteration` if no elements are left.

There's one small catch: one could at times have an iterator without the iterable that generated it and, if iterators don't implement the `__iter__()` method they can't be passed to a `for` loop and are thus quite useless; it is therefore required for an iterator to also have an `__iter__()` method that simply returns the object itself.

That's all there is to it.

Exercise
---

Let's try to reimplement the built-in `range` object. First, observe that calling `range` doesn't give an iterator but an iterable, from which we can generate as many iterators as we want:
```
>>> r = range(3)
>>> list(r) + list(r)
[0, 1, 2, 0, 1, 2]
```
Thus we must implement an iterable and an iterator. Let's start with the former:
```py
class range_iterable:
    def __init__(self, stop):
        self.stop = stop
    def __iter__(self):
        return range_iterator(self.stop)
```
For brevity we don't support `start` or `step` arguments. Now to the second part, the iterator:
```py
class range_iterator:
    def __init__(self, stop):
        self.cur = 0
        self.stop = stop
    def __iter__(self):
        return self
    def __next__(self):
        if self.cur == self.stop:
            raise StopIteration()
        res = self.cur
        self.cur += 1
        return res
```
Does this work?
```
>>> r = range_iterable(3)
>>> list(r) + list(r)
[0, 1, 2, 0, 1, 2]
>>> r = range_iterator(3)
>>> list(r) + list(r)
[0, 1, 2]
```
Yes, it does!
