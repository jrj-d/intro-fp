<style>
.reveal {
  font-size: 32px;
}
.reveal pre {
  width: 100%;
}
</style>

# Returning the current type

---

Common question in Scala:

>I have a type hierarchy … how do I declare a supertype method that returns the “current” type?

Largely based on this [blog post](http://tpolecat.github.io/2015/04/29/f-bounds.html).

----

## Reformulation

Given a trait `Pet`,
> for any expression x with some type `A <: Pet`, ensure that `x.renamed(...)` also has type `A`. To be clear: this is a static guarantee that we want, not a runtime property.

---

## First attempt: subtyping

```tut:silent
trait Pet {
  def name: String
  def renamed(newName: String): Pet
}

case class Fish(name: String, age: Int) extends Pet {
  def renamed(newName: String): Fish = copy(name = newName)
}
```
```tut
val a = Fish("Jimmy", 2)
val b = a.renamed("Bob")
```

----

## Oops

```tut:fail
def doctor[A <: Pet](a: A): A = a.renamed(a.name + ", PhD")
```

----

## Oops #2

```tut:silent
case class Kitty(name: String) extends Pet {
  def renamed(newName: String): Fish = new Fish(newName, 42) // oops
}
```

---

## Second attempt: F-bounded type

```tut:silent
trait Pet[A <: Pet[A]] {
  def name: String
  def renamed(newName: String): A
}

case class Fish(name: String, age: Int) extends Pet[Fish] {
  def renamed(newName: String) = copy(name = newName)
}
```

----

## First victory

```tut
def doctor[A <: Pet[A]](a: A): A = a.renamed(a.name + ", PhD")
val a = Fish("Jimmy", 2)
doctor(a)
```

----

## Oops

```tut:silent
case class Kitty(name: String) extends Pet[Fish] { // oops
  def renamed(newName: String): Fish = new Fish(newName, 42)
}
```

----

## A solution

```tut:silent
trait Pet[A <: Pet[A]] { this: A => // self-type
  def name: String
  def renamed(newName: String): A
}
```

```tut:fail
case class Kitty(name: String) extends Pet[Fish] {
  def renamed(newName: String): Fish = new Fish(newName, 42)
}
```

----

## Oops #2

```tut:silent
class Mammal(val name: String) extends Pet[Mammal] {
  def renamed(newName: String) = new Mammal(newName)
}

class Monkey(name: String) extends Mammal(name)
```
```tut:fail
doctor(new Monkey("Fred"))
```

---

## Third attempt: type class

```tut:reset:silent
trait Pet {
  def name: String
}

trait Rename[A] {
  def rename(a: A, newName: String): A
}

implicit class RenameOps[A](a: A)(implicit ev: Rename[A]) {
  def renamed(newName: String): A = ev.rename(a, newName)
}
```

```tut:silent
object Demo {
	case class Fish(name: String, age: Int) extends Pet

	object Fish {
	  implicit val fishRename = new Rename[Fish] {
	    def rename(a: Fish, newName: String) = a.copy(name = newName)
	  }
	}
}
```

----

## Basic usage

```tut
import Demo._

val a = Fish("Jimmy", 42)

val b = a.renamed("Bob")
```

----

## "Doctor" works

```tut
def doctor[A <: Pet : Rename](a: A): A = a.renamed(a.name + ", PhD")

doctor(Fish("Jimmy", 42))
```

----

## No type mess-up

Impossible to create a `Kitty` that returns a `Fish`, or do the `Mammal trick`.

---

## Summary

|Solution|Return "current" type statically?|Can user mess up?|Method in signature|
|:-------|:-------------------:|:---------------:|:-----------------:|
|Subtyping|no|no|yes|
|F-bounded polymorphism|yes|yes|yes|
|Type class|yes|no|no|

----

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

----

## Libraries

|Library|Method|
|:------|:-----|
|scala.collection|F-bounded polymorphism|
|cats|Type class|
|scalaz|Type class

---

TODO: add links
TODO: search F-bounded vs ad hoc
