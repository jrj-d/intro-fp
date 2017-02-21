<style>
.reveal {
  font-size: 32px;
}
.reveal pre {
  width: 100%;
}
</style>

# Introduction to type classes

---

## Objective of this talk

Avoid

<img src="http://i3.kym-cdn.com/photos/images/newsfeed/000/173/576/Wat8.jpg" width="300">

when presenting functional programming some day.

---

## Type classes & FP

Monads, functors and other functional concepts are all implemented as type classes, because of:
* Haskell predominance on the FP scene,
* expressiveness,
* pimp-my-class pattern,
* static guarantees

---

## Type class

> A type class is a type system construct that supports ad hoc polymorphism.

---

## Polymorphism

> Polymorphism is the provision of a single interface to entities of different types.

* Subtyping: when a name denotes instances of many different classes related by some common superclass.
* Parametric polymorphism: when code is written without mention of any specific type and thus can be used transparently with any number of new types.
* Ad hoc polymorphism: when a function denotes different and potentially heterogeneous implementations depending on a limited range of individually specified types and combinations.

----

## Subtyping

```tut:silent
trait Animal {
	def name: String
}

case class Fish(name: String) extends Animal

def sayHello(animal: Animal): String = s"Hello ${animal.name}"
```
```tut
sayHello(Fish("Nemo"))
```

----

## Parametric polymorphism

Usually called "generics".

```tut
def sayHello[A](a: A): String = s"Hello ${a.toString}"

sayHello(Fish("Nemo"))
```

----

## Ad hoc polymorphism

V1: function overloading, not supported in Scala.

```tut:silent
def add(x: String, y: String): String = x + y
def add(x: Double, y: Double): Double = x + y
```

----

## Ad hoc polymorphism

V2: type class

```tut
def add[T](x: T, y: T)(implicit numeric: Numeric[T]):T = numeric.plus(x, y)

add(3, 4)
add(3.0, 4.0)
```
```tut
def add[T : Numeric](x: T, y: T):T = implicitly[Numeric[T]].plus(x, y)

add(3, 4)
add(3.0, 4.0)
```

---

## (Almost) a real world example

```tut
def sum(ns: List[Int]): Int = ns.fold(0)(_ + _)
def all(bs: List[Boolean]): Boolean = bs.fold(true)(_ && _)
def concat[A](ss: List[List[A]]): List[A] = ss.fold(List.empty[A])(_ ::: _)
```

Let's factorize the commonality.

----

## Requirements

* represent *different* types that behave *similarly* (contract)
* can add capabilities to existing types (pimp my class pattern)
* statically checked by compiler

---

## First idea: subtyping

Bad idea because:
* no reason to have a common supertype with additive capabilities for `Int`, `Boolean` and `List[A]`
* cannot rewrite `Int`!
```scala
trait Int extends Addable
```

---

## Second idea: parametric polymorphism (generics)

```scala
def add[A](x: A, y: A): A = ??? // same code for Int, Boolean and List[A]
```
Impossible!

---

## Third idea: ad hoc polymorphism

This is the one.

---

## Type class

```tut:silent
trait Addable[A] { // type class (the contract)
  def add(a: A, b: A): A
  def zero: A
}

implicit val AddableForInt = new Addable[Int] { // instance
  def add(a: Int, b: Int): Int = a + b
  def zero: Int = 0
}

implicit def addableForList[A] = new Addable[List[A]] { // instance
  def add(a: List[A], b: List[A]): List[A] = a ++ b
  def zero: List[A] = List.empty[A]
}

def genericSum[A](l: List[A])(implicit ev: Addable[A]): A = l.fold(ev.zero)(ev.add)
// ev for evidence
```
```tut
genericSum(List(List("foo", "bar"), List("yep")))
genericSum(List(List(1, 2), List(3, 4)))

```

----

## Pimp my class pattern for free

```tut:silent
implicit class AddableOps[A](a: A)(implicit ev: Addable[A]) {
  def add(b: A): A = ev.add(a, b)
}
```
```tut
4.add(3)
List("foo", "bar").add(List("yep"))
```

---

## Type classes & FP again

```tut:silent
sealed trait Option[+A]
case class Some[A](get: A) extends Option[A]
case object None extends Option[Nothing]

trait Functor[F[_]] {
	def map[A, B](fa: F[A])(f: A => B): F[B]
}

implicit val functorForOption = new Functor[Option] {
	def map[A, B](option: Option[A])(f: A => B): Option[B] = option match {
		case Some(a) => Some(f(a))
		case None => None
	}
}

def incr[F[_], A](fa: F[A])(implicit functor: Functor[F], numeric: Numeric[A]): F[A] = {
	functor.map(fa)(a => numeric.plus(a, numeric.one))
}
```
```tut
incr(Some(4): Option[Int])
incr(None: Option[Double])
incr(Some(4.0): Option[Double])
```

---

## Type class degrades readability

```tut:silent
sealed trait Option[+A]
case class Some[A](get: A) extends Option[A]
case object None extends Option[Nothing]

trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}

implicit val functorForOption = new Functor[Option] {
  def map[A, B](option: Option[A])(f: A => B): Option[B] = option match {
    case Some(a) => Some(f(a))
    case None => None
  }
}
```

How do you know `Option` is a `Functor`?
* Not stated in scaladoc
* Look for the `Functor` implementation

---

## Libraries

|Library|Method|
|:------|:-----|
|scala.collection|F-bounded polymorphism|
|cats|Type class|
|scalaz|Type class

---

TODO: add links
TODO: search F-bounded vs ad hoc
