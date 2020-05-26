# Folders - Folds as Code

**Note: This is not an officially supported Google product.**

Folders is a library that uses *product bifunctors* to define generic,
*delimited* folds and unfolds. In addition to providing predefined folds,
Folders defines a number of generic functions that empower you to define folds
of your own. Let's take a look at some examples:

```clojure
;; The predefined fold, `squash` flattens lists of lists.
(squash '((a b) (c d) (e f)) 3)
;; => (a b c d e f)
;; The predefined unfold `nats` generates the sequence of successors of a natural number.
(nats 100 10)
;; => => (100 101 102 103 104 105 106 107 108 109 110)
;; You can define your own folds using `traverse` and `Traversal` functions
(fold suffixes '() '(Carp rules!) 2)
;; => ((rules!) ())
;; Likewise, you can define unfolds using `accumulate` and `Accumulations` functions.
(reverse (unfold (series +) 0 1 5))
;; => (0 1 2 3 4 5)
```

## Delimited Folds

All of the folds and unfolds in `Folders` are *delimited*; they take an
additional numeric argument that determines the number of times to apply the
fold to the input. This enables you to process only a select segment of a
structure using a fold.

For example, if we only want to sum the first three numbers in a list:

```clojure
(fold (combine +) 0 '(1 2 3 4 5 6 7 8 9 10) 3)
;; => 6 (1 + 2 + 3)
```

### Idempotency

Many folds produce the same result after their input is exhausted, even if
their delimitation is greater than the number of elements in the input, in
other words such folds become *idempotent* on subsequent applications.

For example, let's sum a list of 10 numbers 1000 times:

```clojure
(fold (combine +) 0 '(1 2 3 4 5 6 7 8 9 10) 1000)
;; => 55
```

Not all folds are idempotent. Idempotency depends on the definition of the
fold. For example, the following does not become idempotent:

```clojure
(fold cycle '() '(1 2 3 4) 5)
;; => (1 2 3 4 1 2 3 4 1 2 3 4 1 2 3 4 1 2 3 4)
(fold cycle '() '(1 2 3 4) 10)
;; => (1 2 3 4 1 2 3 4 1 2 3 4 1 2 3 4 1 2 3 4 1 2 3 4 1 2 3 4 1 2 3 4 1 2 3 4 1 2 3 4)
```

Generally speaking, folds tend toward idempotency whenever the operation used
to combine list elements is *monoidal*, that is:

- It has an identity element (like `0` for `+`).
- It is associative.

It's important to note that the operation used in a fold is given by its
traversal and is a *combination* of the binary operation and consumers used to
define the traversal rather than the binary operation on its lonesome.

Traversals over `+` are a good example of this nuance. The traversal given by
`combine` above is defined by `+` and `Consumers.each`. Using a different
consumer, `Consumer.head-idr`, with the same operation yields a traversal that is *not*
idempotent: 

```clojure
(folder (traverse + head-idr) 0 '(1 2 3 4 5 6 7 8 9 10) 1000)
;; => 1000
```

This is because the `head-idr` consumer constantly returns the first value of
the input, `1` without ever consuming the rest of the input. So, the fold never
terminates "naturally" instead terminating only when it has been applied the
number of times designated by the delimitation argument. Calling the above with
`1001` as the delimitation would yield a result of `1001`, `1002` a result of
`1002` and so forth.

## Defining your Own Folds

Two generic higher-order functions, `traverse` and `accumulate`,
list-to-product transformations, called `consumers`, and a special function
`folder` form the underlying machinery for defining folds. You can use
combinations of these functions to define your own folds and unfolds.

## Consumers 

Consumers are functions that transform a list into a product bifunctor. They
are one of the fundamental building blocks for defining folds and "factor out"
or abstract the notion of pattern matching or destructing a list into its
component parts. Typically, a recursive function over some structure proceeds in two steps:

- 1. Inspect the input, checking its structure. 
- 2. Based on the result of (1.) either recursively apply some operation to
     another segment of the input structure *or* cease recurring and return a
computed result.

Consumers handle step (1.) in this process by standardizing the destructuring
or matching on lists into transformation into products.

### A Basic Consumer `Each`

`each` defines what is likely the most prevalent destructure on a list in terms of a product, namely separating a list into two components, its head and tail:

```clojure
(defndynamic each [xs]
  (Folders.Product.fork car cdr xs))
```

> Note: The function `fork` takes a value into a Product by applying two
> functions to it, one produces the left component of the product, the other
> produces the right component.

`each` takes a list and returns a product in which the left component is the
list's head, or element at the front of the list, and the left component is the
list's tail, or the remainder of the list.

### Custom Consumers

To define your own consumer, define a function that takes a list as input and
returns a product of two components. `src/product.carp` contains several
functions for building and manipulating products.

## Traversals and Accumulations

While consumers define ways to match on some input in a recursive context,
traversals, defined using the higher order function `traverse` and
accumulations, defined using the higher order function `accumulate`, describe
an operation to perform on the desctructuresd input returned by a consumer.

Traversals define operations that breakdown input lists, while accumulations
define operations that build up lists based on a seed input, that is, they
define operations for folds and unfolds, respectively.

Both traversals and accumulations use consumers to manipulate the list
structures they either build up or tear down.

### Traversals

You can define your own traversals using the `traverse` function. `traverse`
requires two arguments to define a traversal:

- a binary function `f` that will be applied to the results of the consumer `t` at each recursive step.
- a consumer `t` that breaks down an input list into a product of two components.

Different combinations of consumers and binary functions yield different
traversals (the [prior section on idempotency](#idempotency) gives one example
of this.  

For example, we can define a simple traversal `sum` as the combination of `+`
and the `each` consumer, which will apply the operation to each value in the
list:

```clojure
(traverse + each)
```

Some traversals are also higher-order, allowing for either variable binary
operations or consumers, such as `combine`, which takes a binary operation as
an argument:

```clojure
(defndynamic combine [m] (Folders.Traversals.traverse m Folders.Consumers.each))
```

### Accumulations

Accumulations, like traversals, use consumers to manipulate lists in terms of
products, but instead produce or build lists from a seed value instead of
reducing them. Accumulations also take a binary operation to perform on the
consumer results, but additionally take a "list construction" operation to use
when accumulating results.

Just like traversals, different combinations of binary operation, consumer, and
list construction strategies will yield different accumulations.

A simple accumulation, successors, is given by combining `+`, the `head-idr`
consumer, and `cons`:

```clojure
(accumulate + head-idr cons)
```

Accumulations can also be higher order; "successors" may be defined in terms of
the more generic `series` accumulation:

```clojure
(defndynamic series [m]
  (Folders.Accumulations.accumulate m Folders.Consumers.head-idr cons))
```

### Folder

While consumers take care of pattern matching on input and traversals or
accumulations handle performing some operation on the input, the special
function `folder` handles delimited recursion, taking care of recursively
calling the operations you've defined.

You can use `folder` to generically define a variety of folds. For example, the
predefined fold `squash` is defined as follows:

```clojure
(defndynamic squash [xs n]
  (Folders.folder Folders.Traversals.concat '() xs n 0))
```

And the predefined unfold `nats` is defined as:

```clojure
(defndynamic nats [start n]
  (reverse (if (not (list? seed))
               (Folders.folder accumulation (list seed) step limit 0)
               (Folders.folder accumulation seed step limit 0))))
```

[^1]: A *product bifunctor* is a product (a combination of two values) that
additionally satisfies functor laws.
