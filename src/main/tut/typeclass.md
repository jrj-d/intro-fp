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

## Type class

> A type class is a type system construct that supports ad hoc polymorphism.

---

## Polymorphism

> Polymorphism is the provision of a single interface to entities of different types.

* Subtyping: when a name denotes instances of many different classes related by some common superclass.
* Parametric polymorphism: when code is written without mention of any specific type and thus can be used transparently with any number of new types.
* Ad hoc polymorphism: when a function denotes different and potentially heterogeneous implementations depending on a limited range of individually specified types and combinations.

---

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

---

## Parametric polymorphism

Usually called "generics".

```tut
def sayHello[A](a: A): String = s"Hello ${a.toString}"

sayHello(Fish("Nemo"))
```

---

## Ad hoc polymorphism

Function overloading, not supported in Scala.

```tut:silent
def add(x: String, y: String): String = x + y
def add(x: Double, y: Double): Double = x + y
```

---

## Ad hoc polymorphism

How is it done in Scala then? Type class

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

Let's factorize the commonality. We'd like:
```scala
def genericSum[A](l: List[A]): A // for some adequate A
```

---

## Requirements

```scala
def genericSum[A](l: List[A]): A // for some adequate A
```

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

## Wrapper

```tut:silent
trait Addable[A] {
  def add(b: Addable[A]): Addable[A]
  def get: A
}

case class AddableList[A](a: List[A]) extends Addable[List[A]] {
  def add(b: Addable[List[A]]): AddableList[A] = AddableList(a ++ b.get)
  def get: List[A] = a
}

def genericSum[A](l: List[Addable[A]]): A = l.reduce(_.add(_)).get
```
```tut
genericSum(List(AddableList(List("foo", "bar")), AddableList(List("yep"))))
```
Cumbersome and unsafe (reduce fails when list is empty).

---

## Type class

```tut:silent
trait Addable[A] { // type class (the contract)
  def add(a: A, b: A): A
  def zero: A
}

implicit val addableForInt = new Addable[Int] { // instance
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
genericSum(List(5, 3, 4))
```
```tut:fail
genericSum(List(5L, 3L, 4L))
```

---

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

## Type class degrades readability

Code obfuscation:
```scala
def genericSum[F[_] : Foldable, A : Addable](l: F[A]): A =
    Foldable[F].foldLeft(l, Addable[A].zero)(Addable[A].add)
```
Not a real problem, it just takes some getting used to.

---

## Type class degrades readability #2

How do you know `List[A]` is `Addable`?

Look for the implicit instance. But:
* you have to track implicits in your code
* methods from pimp-my-class pattern don't appear in scaladoc
* you'd better use an IDE

Example with [cats](http://typelevel.org/cats/).

---

## Boilerplate

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

implicit class AddableOps[A](a: A)(implicit ev: Addable[A]) { // boilerplate
  def add(b: A): A = ev.add(a, b)
}
```

---

## Simulacrum

```scala
import simulacrum._

@typeclass trait Semigroup[A] {
  @op("|+|") def append(x: A, y: A): A
}
```
Generated code:
```scala
trait Semigroup[A] {
  def append(x: A, y: A): A
}

object Semigroup {
  def apply[A](implicit instance: Semigroup[A]): Semigroup[A] = instance

  trait Ops[A] {
    def typeClassInstance: Semigroup[A]
    def self: A
    def |+|(y: A): A = typeClassInstance.append(self, y)
  }

  trait ToSemigroupOps {
    implicit def toSemigroupOps[A](target: A)(implicit tc: Semigroup[A]): Ops[A] = new Ops[A] {
      val self = target
      val typeClassInstance = tc
    }
  }

  object nonInheritedOps extends ToSemigroupOps

  trait AllOps[A] extends Ops[A] {
    def typeClassInstance: Semigroup[A]
  }

  object ops {
    implicit def toAllSemigroupOps[A](target: A)(implicit tc: Semigroup[A]): AllOps[A] = new AllOps[A] {
      val self = target
      val typeClassInstance = tc
    }
  }
}
```

---

## Simulacrum #2

```scala
implicit val semigroupInt: Semigroup[Int] = new Semigroup[Int] {
  def append(x: Int, y: Int) = x + y
}

import Semigroup.ops._
1 |+| 2 // 3
```
[Simulacrum](https://github.com/mpilquist/simulacrum).

---

## Functional programming libraries

|Libraries|Method|
|:------|:-----|
|[scala.collection](http://docs.scala-lang.org/overviews/collections/overview.html)|F-bounded polymorphism|
|[cats](http://typelevel.org/cats/)|Type class|
|[scalaz](https://github.com/scalaz/scalaz)|Type class|

Good introduction to functional programming in Scala: [herding cats](http://eed3si9n.com/herding-cats/).
More about [typeclasses](http://danielwestheide.com/blog/2013/02/06/the-neophytes-guide-to-scala-part-12-type-classes.html).

---

# Questions
