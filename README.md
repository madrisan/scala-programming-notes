# Functional Programming in Scala

## Collections

Scala has a uite rich hierarchy of collection classes.

![Scala Collections](http://docs.scala-lang.org/resources/images/collections.png "Scala Collection Hierarchy")

All collection types share a common set of general methods. The core methods are we are interested here are: __map__, __flatMap__, __filter__.

*Idealized Implementation of __map__ on Lists*
```scala
abstract class List[+T] {
  def map[U](f: T => U): List[U] = this match {
    case x :: xs => f(x) :: xs.map(f)
    case Nil => Nil
  }
}
```
*Idealized Implementation of __flatMap__ on Lists*
```scala
abstract class List[+T] {
  def flatMap[U](f: T => List[U]): List[U] = this match {
    case x :: xs => f(x) ++ xs.flatMap(f)
    case Nil => Nil
  }
}
```

*Idealized Implementation of __filter__ on Lists*
```scala
abstract class List[+T] {
  def filter(p: T => Boolean): List[T] = this match {
    case x :: xs =>
      if (p(x)) x :: xs.filter(p) else xs.filter(p)
    case Nil => Nil
  }
}
```

Note that the implementation and type of these methods in the Scala library are different in order to make them apply to _arbitrary collections_ and make them _tail-recursive_ on lists (to give us something that works in contant stack space).

## For-Expressions

For-Expressions are useful because they gice you a simpler notation for something that comes down to combinations of these core methods, _map_, _flatMap_, and _filter_.

The Scala compiler translates for-expressions in terms of _map_, _flatMap_ and _withFilter_, a lazy variant of _filter_, that does not produce an intermediate list, but instead filters the following _map_ or _flatMap_ function application.
 
Here is the translation scheme used by the compiler:

```scala
for (x <- e1) yield e2                         -->   e1.map(x => e2)
for (x <- e1 if f; s) yield e2                 -->   for (x <- e1.withFilter(x => f); s) yield e2
for (x <- e1.withFilter(x => f); s) yield e2   -->   e1.flatMap(x => for (y <- e2; s) yield e3)
```
where _f_ is a filter and _s_ is a (potentially empty) sequence of generators and filters.

#### Example of a for-expression translation into higher-order functions

Take the for-expression that computed pairs whose sum is prime:
```scala
for {
  i <- 1 until n
  j <- 1 until i
  if isPrime(i + j)
} yield (i, j)
```
Applying the translation scheme to this expression gives:
```scala
(1 until n).flatMap(i =>
  (1 until i).withFilter(j => isPrime(i+j))
    .map(j => (i, j)))
```

The translation of for is _not_ limited to lists or sequences, or even collections.
It is based solely on the presence of the methods _map_, _flatMap_ and _withFilter_.
This lets you use the for syntax for your own types as well – you must only define _map_, _flatMap_ and _withFilter_ for these types.

There are many types for which this is useful: arrays, iterators, databases, XML data, optional values, parsers, etc.

This is the basis of the Scala data base connection frameworks
[ScalaQuery](http://scalaquery.org/)
(an API / DSL (domain specific language) built on top of
[JDBC](http://java.sun.com/products/jdbc/overview.html) for accessing relational databases in Scala) and
[Slick](http://slick.lightbend.com/) (a modern database query and access library for Scala).

#### Example - Functional Random Generators

This is an example of a Integer number generator, using the class _java.util.Random_
```scala
import java.util.Random

val integers = new Generator[Int] {
  val rand = new java.util.Random
  def generate = rand.nextInt()
}
```

_Question_: What is a systematic way to get random values for other domains, such as booleans, strings, pairs and tuples, lists, sets, trees?

Let’s define a trait _Generator[T]_ that generates random values of type _T_, and the _map_ and _flatMap_ methods.
```scala
trait Generator[+T] {
  self =>       // an alias for ”this”

  def generate: T

  def map[S](f: T => S): Generator[S] = new Generator[S] {
    def generate = f(self.generate)
  }
  
  def flatMap[S](f: T => Generator[S]): Generator[S] = new Generator[S] {
    def generate = f(self.generate).generate
  }
```

The _booleans Generator_
```scala
val booleans = for (x <- integers) yield x > 0
```
expands to
```scala
val booleans = integers map { x => x > 0 }

val booleans = new Generator[Boolean] {
  def generate = (x: Int => x > 0)(integers.generate)
}

val booleans = new Generator[Boolean] {
  def generate = integers.generate > 0
}
```
The _pairs_ Integer Generator
```scala
val pairs = new Generator[(Int, Int)] {
  def generate = (integers.generate, integers.generate)
}
```
can be generalized and expand to
```scala
def pairs[T, U](t: Generator[T], u: Generator[U]) = t flatMap {
  x => u map { y => (x, y) }
}

def pairs[T, U](t: Generator[T], u: Generator[U]) = t flatMap {
  x => new Generator[(T, U)] { def generate = (x, u.generate) }
}

def pairs[T, U](t: Generator[T], u: Generator[U]) = new Generator[(T, U)] {
  def generate = (new Generator[(T, U)] {
    def generate = (t.generate, u.generate)
  }).generate }
```

#### Generator Examples

```scala
def single[T](x: T): Generator[T] = new Generator[T] {
  def generate = x
}

def choose(lo: Int, hi: Int): Generator[Int] =
  for (x <- integers) yield lo + x % (hi - lo)

def oneOf[T](xs: T*): Generator[T] =
  for (idx <- choose(0, xs.length)) yield xs(idx)
```
A _List_ Generator

A list is either an empty list or a non-empty list.
```scala
def lists: Generator[List[Int]] = for {
  isEmpty <- booleans
  list <- if (isEmpty) emptyLists else nonEmptyLists
} yield list

def emptyLists = single(Nil)

def nonEmptyLists = for {
  head <- integers
  tail <- lists
} yield head :: tail
```

A Tree Generator
```scala
trait Tree
case class Inner(left: Tree, right: Tree) extends Tree
case class Leaf(x: Int) extends Tree
```

Random Test Function
Using generators, we can write a random test function:
```scala
def test[T](g: Generator[T], numTimes: Int = 100)
  (test: T => Boolean): Unit = {
    for (i <- 0 until numTimes) {
      val value = g.generate
      assert(test(value), ”test failed for ”+value)
    }
    println(”passed ”+numTimes+” tests”)
  }
```

## Monads

A monad *M* is a parametric type *M[T]* with two operations, `flatMap` and `unit`
```scala
trait M[T] {
  def flatMap[U](f: T => M[U]): M[U]
}

def unit[T](x: T): M[T]
```
that have to satisfy the following laws:

*Associativity*:
```scala
m flatMap f flatMap g == m flatMap (x => f(x) flatMap g)
```
*Left unit*:
```scala
unit(x) flatMap f == f(x)
```
*Right unit*:
```scala
m flatMap unit == m
```

#### Examples of Monads

- *List* is a monad with `unit(x) = List(x)`
- *Set* is monad with `unit(x) = Set(x)`
- *Option* is a monad with `unit(x) = Some(x)`
- *Generator* is a monad with `unit(x) = single(x)`

##### Let's demonstrate for example that Option is a monad.

Here’s *flatMap* for *Option*:
```scala
abstract class Option[+T] {
  def flatMap[U](f: T => Option[U]): Option[U] = this match {
    case Some(x) => f(x)
    case None => None
  }
}
```

*Associativity Law*
```
    m flatMap f flatMap g

==  m match { case Some(x) => f(x) case None => None }
      match { case Some(y) => g(y) case None => None }
==  m match {
      case Some(x) =>
        f(x) match { case Some(y) => g(y) case None => None }
      case None =>
        None match { case Some(y) => g(y) case None => None }
      }
==  m match {
      case Some(x) =>
        f(x) match { case Some(y) => g(y) case None => None }
      case None => None
    }
==  m match {
      case Some(x) => f(x) flatMap g
      case None => None
    }
==  m flatMap (x => f(x) flatMap g)
QED
```
*Left Unit Law*
```
    Some(x) flatMap f
  
==  Some(x) match {
      case Some(x) => f(x)
      case None => None
    }
    
==  f(x)
QED
```
*Right Unit Law*
```
    m flatMap Some
    
==  m match {
      case Some(x) => Some(x)
      case None => None
    }
    
==  m
QED
```

#### Monad and For-Expressions

Monad-typed expressions are typically written as for expressions.



