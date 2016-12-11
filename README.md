# Functional Programming in Scala

## Collections

All collection types share a common set of general methods. The core methods are: __map__, __flatMap__, __filter__.

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

The Scala compiler translates for-expressions in terms of _map_, _flatMap_ and a lazy variant of _filter_. Here is the translation scheme used by the compiler:

```scala
for (x <- e1) yield e2                         -->   e1.map(x => e2)
for (x <- e1 if f; s) yield e2                 -->   for (x <- e1.withFilter(x => f); s) yield e2
for (x <- e1.withFilter(x => f); s) yield e2   -->   e1.flatMap(x => for (y <- e2; s) yield e3)
```
where _f_ is a filter and _s_ is a (potentially empty) sequence of generators and filters.

You can think of _withFilter_ as a variant of filter that does not produce an intermediate list, but instead filters the following map or _flatMap_ function application.

### Example

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



