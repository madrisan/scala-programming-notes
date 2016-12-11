# Functional Programming in Scala

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



