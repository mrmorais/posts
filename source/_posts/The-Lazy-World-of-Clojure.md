---
title: (English) The Lazy World of Clojure
date: 2019-10-22 12:48:05
tags:
---

# (English) The Lazy World of Clojure
08 de outubro de 2019

In functional programming context laziness means delaying the execution of a code or, putting in other words: you only compute (realize) a code when you need the result.

Clojure is not a lazy language (like Haskell is) but it supports lazily evaluated sequences and most core functions uses lazy-seq under the hood, so if you're programming in Clojure you're using lazy-seq (as we'll gonna call it from now.)

There are some benefits of using lazy-seq: the first one is that we can represent infinite sequences (fibonacci, odd and even, prime numbers.) Another good reason to use lazy-seqs is that it does caching and thunking with the results, as we'll see next.

A good way to think is that a lazy-seq only stores a recipe and a state of the sequence, so it don't have to perform all of them. It's very close to how functions works. Until a function got called the body is not evaluated, even because it only have the recipe but the ingredients (the parameters) are arriving yet.

```
(def ten (+ 5 5)) ; is evaluated on execution
(defn ten [] (+ 5 5)) ; body is evaluated later
```

In the first example, the sum must be evaluated what is not true for the second one.

Let's get into lazy-seq by Fibonacci's sequence example:

```
(defn fib
  ([] (fib 0 1))]
  ([x y] (lazy-seq (cons x (fib y (+ x y))))))
```

The lazy-seq forces to postpone the evaluation of `cons`. `cons` returns a seq where the first param is the first element and the second param is the rest. For instance:

```
(cons 1 [2 3])
; (1 2 3)
```

So, while taking our fibonacci sequence it would calculate far as necessary instead perform all recurrences (what may take a while). A interesting experience is to add a Thread/sleep to that fib function to see how lazy-seq caches the results.

```
(defn fib
  ([] (fib 0 1))
  ([x y] (Thread/sleep 1000) (lazy-seq (cons x (fib y (+ x y))))))
```

You'll notice that at the first time it will take the n * 1000 milliseconds to respond, but when call it again it will be immediate.

```
(def a (fib))
user => (take 10 a)
; (0 1 1 2 3 5 8 13 21 34) - 10 seconds after...
user => (take 10 a)
; (0 1 1 2 3 5 8 13 21 34) - immediately
```

Lazy-seq is a very useful Clojure feature, there's certainly a lot more to read and learn about it, but for now I hope that you enjoyed what I had to talk about this subject.