# Methods vs Free Functions

Reference: `lib/src/sha256.rs` — `Hash::matches_target`

## The core question

Is a method fundamentally different from a function, or is it just syntax sugar?

---

## Java — methods are fundamental to the object

In Java, a method *belongs* to a class. You cannot define behaviour outside of a class.
There are no free functions — everything must live inside a class definition.

```java
public class Hash {
    private BigInteger value;

    // this method is baked into the class — it IS part of Hash
    public boolean matchesTarget(BigInteger target) {
        return this.value.compareTo(target) <= 0;
    }
}
```

- `this` is implicit and injected by the runtime
- The method dispatch goes through the object's vtable (for virtual methods)
- You cannot separate the struct from its methods — they are one thing
- A standalone `matchesTarget(Hash h, BigInteger t)` outside a class is illegal

**Consequence:** in Java you often see utility classes like `Collections`, `Arrays`, `Math`
that are just bags of static methods — a workaround for "everything must be in a class."

---

## Python — methods are functions, but with a twist

Python is more honest about it. A method is literally just a function that happens to live
in a class body. `self` is explicit and is just a regular first argument.

```python
class Hash:
    def matches_target(self, target):  # self is just a normal parameter
        return self.value <= target
```

You can even call it like a free function:
```python
h = Hash(some_value)
Hash.matches_target(h, target)  # identical to h.matches_target(target)
```

Python makes the illusion visible — `h.matches_target(target)` is literally just
`Hash.matches_target(h, target)` under the hood. The runtime does a lookup on the class
and passes the instance as the first argument.

**But:** Python has no compile-time checks. The method is looked up at runtime via
`__dict__`. If the method doesn't exist, you get `AttributeError` when that line runs.
Duck typing — the check is deferred until execution.

---

## Rust — methods are free functions with syntactic sugar

Rust is explicit like Python (`self` is a real parameter) but verified like Java (at
compile time). The `impl` block is just a namespace for functions — the struct and its
behaviour are genuinely separate things.

```rust
pub struct Hash(U256);  // data — just a struct, no behaviour

impl Hash {             // behaviour — attached separately
    pub fn matches_target(&self, target: U256) -> bool {
        self.0 <= target
    }
}
```

The compiler translates `hash.matches_target(target)` into:
```rust
Hash::matches_target(&hash, target)  // exactly equivalent
```

You could write `matches_target` as a free function and nothing would change mechanically:
```rust
pub fn matches_target(hash: &Hash, target: U256) -> bool {
    hash.0 <= target
}
```

The only reason to prefer the method form is ergonomics:
- `hash.matches_target(target)` reads naturally
- Methods are namespaced under `Hash::` — no name collisions
- IDE autocomplete works when you type `hash.`

---

## Side-by-side comparison

| | Java | Python | Rust |
|---|---|---|---|
| Can you have free functions? | No — must be in a class | Yes | Yes |
| Is `self`/`this` explicit? | No — implicit `this` | Yes — explicit `self` | Yes — explicit `self` |
| Are methods just functions? | No — fundamentally different | Yes — sugar over function call | Yes — sugar over function call |
| When is dispatch checked? | Compile time (mostly) | Runtime | Compile time |
| Struct and behaviour separated? | No — one class definition | No — one class definition | Yes — `struct` and `impl` are separate |

---

## Why Rust separates struct and impl

This separation enables two powerful patterns:

**1. Implement a trait for a type you didn't write:**
```rust
// you can add behaviour to Vec<u8> by implementing your own trait for it
impl MyTrait for Vec<u8> { ... }
```
In Java/Python you'd have to subclass or wrap it.

**2. Multiple impl blocks:**
```rust
impl Hash { ... }                    // core methods
impl fmt::Display for Hash { ... }   // display behaviour added separately
impl PartialOrd for Hash { ... }     // comparison behaviour added separately
```

Behaviour can be spread across impl blocks and even across files. The struct definition
stays clean — just data.
