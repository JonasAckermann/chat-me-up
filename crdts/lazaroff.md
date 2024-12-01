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

### 1. Last Write Wins Register

Note: Possibly location data as is?

> LWW Registers, as the name suggests, simply overwrite their current value with the last value written. They determine which write occurred last using timestamps, represented here by integers that increment whenever the value is updated. Here’s the algorithm:

> * If the received timestamp is less than the local timestamp, the register doesn’t change its state.
> * If the received timestamp is greater than the local timestamp, the register overwrites its local value with the received value. It also stores the received timestamp and some sort of identifier unique to the peer that last wrote the value (the peer ID).
> * Ties are broken by comparing the local peer ID to the peer ID in the received state.

> You might ask: why not use the actual time? Unfortunately, accurately syncing clocks between two computers is an extremely hard problem. Using incrementing integers like this is one simple version of a logical clock, which captures the order of events relative to each other rather than to the “wall clock”.

ChatGPT:

> Below is an immutable Scala implementation of the given LWWRegister class. The class is designed to follow immutability principles, ensuring that state changes return new instances rather than modifying the existing object:

NOTE: Not tested for compilation, would style it slightly differently but it's currently trying to be similar to the original.

```
case class LWWRegister[T](id: String, state: (String, Long, T)) {

  // Getter for the value
  def value: T = state._3

  /**
   * Creates a new LWWRegister with an updated state.
   * The peer ID is set to the local ID, the timestamp is incremented by 1, and the value is updated.
   */
  def set(value: T): LWWRegister[T] = {
    val (peer, timestamp, _) = state
    copy(state = (id, timestamp + 1, value))
  }

  /**
   * Merges the current state with the provided state.
   * Returns a new LWWRegister with the merged state.
   */
  def merge(remoteState: (String, Long, T)): LWWRegister[T] = {
    val (remotePeer, remoteTimestamp, remoteValue) = remoteState
    val (localPeer, localTimestamp, localValue) = state

    // Determine which state to keep based on timestamp and peer ID
    if (localTimestamp > remoteTimestamp ||
        (localTimestamp == remoteTimestamp && localPeer > remotePeer)) {
      this // Keep the current state
    } else {
      copy(state = remoteState) // Overwrite with the remote state
    }
  }
}
```
**Explanation:**

* Immutability: \
  The class is implemented as a Scala case class to provide immutability and boilerplate code for copy, equals, and hashCode.
  Instead of modifying state directly, the set and merge methods return new instances of `LWWRegister` with updated state.
* State Representation: \
The state is represented as a tuple `(String, Long, T)` to match the structure `[peer: string, timestamp: number, value: T]`.
* `set` Method: \
Creates a new instance with the id as the peer, the timestamp incremented by 1, and the updated value.
* `merge` Method: \
Compares timestamps and peer IDs to determine whether to keep the local state or overwrite it with the remote state.
Returns the current instance if the local state is preferred or a new instance with the remote state otherwise.

### 2. Last Write Wins Map 
> Do you see the trick? From our application’s perspective, the LWW Map just holds normal values — but it actually holds LWW Registers. When we look at the full state, each key’s state is the state of the LWW Register at that key 

> Composition lets us combine primitive CRDTs into more complex ones. When it’s time to merge, all the parent does is pass slices of incoming state to the appropriate child’s merge function

> LWW Map merge function is simple: iterate through each key and hand off the incoming state at that key to the corresponding LWW Register to merge.

**ChatGPT:**
> To translate the provided TypeScript type alias for an object with string keys and values of a generic type `T` into immutable Scala, you can use a `Map[String, T]`. Here's the equivalent definition:

```
type Value[T] = Map[String, T]
```
**Explanation:**
> * Map in Scala:
The Scala `Map` type represents an immutable collection of key-value pairs by default (unless explicitly using `mutable.Map`).
> * The keys in `Map[String, T]` are strings, corresponding to the string keys in the TypeScript object.
> * The values are of type `T`, making it generic and aligning with the original definition.

> The TypeScript type definition for `State<T>` maps strings to the state property of an `LWWRegister` object, where the type of state is a tuple. Here's how you can represent this as an immutable type in Scala:

> Assuming LWWRegister is defined with a state field of type (String, Long, T) in Scala, the equivalent would be:

```
type State[T] = Map[String, (String, Long, T)]
```
**Explanation:**
> 1. Key as String:
 Just like in the TypeScript version, the keys are strings, represented as `String` in Scala.
> 2. Value as Tuple:
The `state` field in `LWWRegister` is a tuple in Scala: `(String, Long, T)`. This aligns with the structure described in TypeScript.
> 3. Generic Handling of T | null:
In TypeScript, `T | null` allows the value type to be `T` or `null`. In Scala, `Option[T]` is idiomatic.
If nullability needs to be supported, replace `T` in the tuple with `Option[T]` to account for this:
```
type State[T] = Map[String, (String, Long, Option[T])]
```
NOTE: Don't think we need nullability in the simplest case.
NOTE: Actually, I was wrong. If peer deletes, delete.

**ChatGPT**

NOTE: I added comments based on the original article's explanations to the individual methods.

```
case class LWWMap[T](id: String, private val data: Map[String, LWWRegister[Option[T]]]) {

  /**
   * Constructs an `LWWMap` from an ID and an initial state.
   */
  def this(id: String, state: Map[String, (String, Long, Option[T])]) = {
    this(
      id,
      state.map { case (key, register) =>
        key -> LWWRegister(id, register)
      }
    )
  }

  /**
   * Retrieves the current value map.
   * Only keys with non-null values are included.
   */
  def value: Map[String, T] =
    data.collect {
      case (key, register) if register.value.isDefined =>
        key -> register.value.get
    }

  /**
   * Retrieves the current state map.
   */
  def state: Map[String, (String, Long, Option[T])] =
    data.map { case (key, register) =>
      key -> register.state
    }

  /**
   * Checks if a key exists and its value is not null.
   */
  def has(key: String): Boolean =
    data.get(key).exists(_.value.isDefined)

  /**
   * Retrieves the value for a given key, if it exists.
   */
  def get(key: String): Option[T] =
    data.get(key).flatMap(_.value)

  /**
   * Adds or updates a key with a new value, returning a new LWWMap.
   */
  def set(key: String, value: T): LWWMap[T] = {
    val updatedData = data.get(key) match {
      case Some(register) => data.updated(key, register.set(Some(value)))
      case None           => data.updated(key, LWWRegister(id, (id, 1L, Some(value))))
    }
    copy(data = updatedData)
  }

  /**
   * Deletes a key by setting its value to null, returning a new LWWMap.
   */
  def delete(key: String): LWWMap[T] =
    data.get(key) match {
      case Some(register) =>
        copy(data = data.updated(key, register.set(None)))
      case None =>
        this // No change if the key doesn't exist
    }

  /**
   * Merges the current LWWMap with a new state, returning a new LWWMap.
   */
  def merge(state: Map[String, (String, Long, Option[T])]): LWWMap[T] = {
    val mergedData = state.foldLeft(data) {
      case (currentData, (key, remoteState)) =>
        currentData.get(key) match {
          case Some(localRegister) =>
            currentData.updated(key, localRegister.merge(remoteState))
          case None =>
            currentData.updated(key, LWWRegister(id, remoteState))
        }
    }
    copy(data = mergedData)
  }
}
```
