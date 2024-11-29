# Jake Lazaroff on CRDTs

Notes on and playing around with the excellent https://jakelazaroff.com/words/an-interactive-intro-to-crdts/

> * A value, T. This is the part the rest of our program cares about. The entire point of the CRDT is to reliably sync the value between peers.

> * A state, S. This is the metadata needed for peers to agree on the same value. To update other peers, the whole state is serialized and sent to them.

> * A merge function. This is a function that takes some state (probably received from another peer) and merges it with the local state.

## Rust and Scala interfaces

The following is adapted from ChatGPT responses:


> Can you transform this TypeScript snippet into their Scala and Rust equivalents?

```
interface CRDT<T, S> {
  value: T;
  state: S;
  merge(state: S): void;
}
``` 


> ChatGPT
Certainly! Here's how your TypeScript snippet can be translated into Scala and Rust equivalents:

### Scala
In Scala, you can define a similar construct using a trait to represent the interface.

```
trait CRDT[T, S] {
  var value: T
  var state: S

  def merge(state: S): Unit
}
```

**Explanation:**

* `trait` is used to define an interface-like structure in Scala.
* `var` is used to allow mutable fields for value and state.
* The `merge` method takes a parameter of type S and returns `Unit` (equivalent to void in TypeScript).



### Rust
In Rust, you would use a trait to represent the interface and use generics for T and S.

```
pub trait CRDT<T, S> {
    fn value(&self) -> &T;
    fn state(&self) -> &S;
    fn merge(&mut self, state: S);
}
```
**Explanation:**
* `trait` is used to define the behavior.
value and state are represented as getter methods since Rust traits don't have fields.
* `&self` denotes that the method doesn't mutate the instance, while `&mut` self indicates mutable access for the merge method.
* `fn` merge consumes an argument of type S.

If you wanted to implement this trait for a specific struct, you could do so as follows:
```
struct MyCRDT<T, S> {
    value: T,
    state: S,
}

impl<T, S> CRDT<T, S> for MyCRDT<T, S> {
    fn value(&self) -> &T {
        &self.value
    }

    fn state(&self) -> &S {
        &self.state
    }

    fn merge(&mut self, state: S) {
        self.state = state; // Example logic
    }
}
```

### Immutable Scala Version 

> Can you change the scala example to not use mutable vars?


> Certainly! To avoid mutable vars in the Scala example, you can use vals for immutability and return updated instances of the CRDT in methods that would otherwise mutate state. Here's the revised Scala version:

```
trait CRDT[T, S] {
  def value: T
  def state: S

  def merge(state: S): CRDT[T, S]
}
```
**Explanation:**
* `value` and `state` are defined as `def` instead of var or val, meaning they are abstract methods in the trait that must be implemented by concrete classes or case classes.
* `merge` returns a new instance of the CRDT with the updated state, adhering to immutability.

**Implementation Example**
To implement this trait, you can use a case class, which is naturally immutable in Scala:

```
case class MyCRDT[T, S](value: T, state: S) extends CRDT[T, S] {
  override def merge(newState: S): CRDT[T, S] = 
    copy(state = newState)
}
```

**Explanation:**

* `MyCRDT` is a case class implementing the CRDT trait.
* The `merge` method uses the copy method to create a new instance with the updated state, keeping the existing value.

## Checking Properties in Scala

From the post: 
> * Commutativity: states can be merged in any order; A ∨ B = B ∨ A. If Alice and Bob exchange states, they can each merge the other’s state into their own and arrive at the same result.
> * Associativity: when merging three (or more) states, it doesn’t matter which are merged first; (A ∨ B) ∨ C = A ∨ (B ∨ C). If Alice receives states from both Bob and Carol, she can merge them into her own state in any order and the result will be the same.4
> * Idempotence: merging a state with itself doesn’t change the state; A ∨ A = A. If Alice merges her own state with itself, the result will be the same state she started with.

**ChatGPT**
> Here’s how you can express these three mathematical properties in Scala code. The code assumes a trait or case class representing a State with a merge method (∨ operation) that satisfies these properties.

NOTE: Have not checked if these compile.

### 1. Commutativity
```
def commutativity[A](a: A, b: A)(merge: (A, A) => A): Boolean = {
  merge(a, b) == merge(b, a)
}

// Example usage:
// Assuming `State` has a `merge` method
case class State(value: Int) {
  def merge(other: State): State = State(this.value + other.value) // Example merge logic
}

val state1 = State(5)
val state2 = State(3)

assert(commutativity(state1, state2)((a, b) => a.merge(b)))
``` 
### 2. Associativity
```
def associativity[A](a: A, b: A, c: A)(merge: (A, A) => A): Boolean = {
  merge(merge(a, b), c) == merge(a, merge(b, c))
}

// Example usage:
assert(associativity(state1, state2, State(7))((a, b) => a.merge(b)))
```
### 3. Idempotence
```
def idempotence[A](a: A)(merge: (A, A) => A): Boolean = {
  merge(a, a) == a
}

// Example usage:
assert(idempotence(state1)(a => a.merge(a)))
``` 
**Explanation** 
* Commutativity:
The commutativity function checks if merging a and b is the same regardless of the order.
* Associativity:
The associativity function checks if merging three states in different groupings produces the same result.
* Idempotence:
The idempotence function ensures that merging a state with itself doesn’t change the state.


**Extensibility**

If you have a specific merge logic, you can implement it within the merge method of your state class or as a standalone function passed into these tests.

## Existing CRDTs
* Last Write Wins Register -> Possibly location data as is?
* Last Write Wins Map
* 

