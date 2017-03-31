<style>
.reveal {
  font-size: 32px;
}
.reveal pre {
  width: 100%;
}
.reveal section img {
  margin: 15px 0px;
  background: transparent;
  border: 0px;
  box-shadow: 0 0 0px rgba(0, 0, 0, 0.15);
}
</style>

# Effects in functional programming

---

## Last lecture in 2 min

FP is cool because:
* readability
* more compile-time checks
* lazy evaluation, parallelism, memoization

FP requires *pure functions*.

---

## Rethink the way you code

* Higher-order functions instead of loops
* Recursion instead of (some) while loops
* Immutable data for safety

## What about those?

* Access a configuration
* Maintain a state
* Query a database
* write to a file

All those are impure: they contain side effects!
