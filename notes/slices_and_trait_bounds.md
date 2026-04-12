# Slices and Trait Bounds on Fields

---

## `&[T]` — slice references

Reference: `lib/src/util.rs`

```rust
pub fn calculate(transactions: &[Transaction]) -> MerkleRoot {
```

`&[Transaction]` is a **reference to a slice** — a reference to a contiguous sequence of
`Transaction` values. The transactions themselves are values, not references.

Under the hood it is just two words:
```
&[Transaction]  =  { ptr: *const Transaction, len: usize }
```

**Contiguous is a guarantee, not just "iterable".**  
`[T]` is defined as a contiguous block of memory — like a C array with a length. It is not
a general iterable like a linked list or hash map. Elements sit next to each other in memory.

**Why `&[T]` instead of `&Vec<T>`:**

`&[T]` is more flexible. It accepts any contiguous source via automatic coercion:

```rust
let v: Vec<Transaction> = vec![...];
calculate(&v);           // Vec<T> coerces to &[T]

let a: [Transaction; 3] = [...];
calculate(&a);           // fixed array also coerces to &[T]
```

If you wrote `&Vec<Transaction>` you'd only accept a `Vec`. Using `&[T]` in function
signatures is idiomatic Rust — it means "a read-only contiguous view, I don't care how
you allocated it."

---

## Derive macros require fields to implement the same trait

Reference: `lib/src/types.rs`

```rust
#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct Block {
    pub header: BlockHeader,
    pub transactions: Vec<Transaction>,
}
```

`#[derive(Clone)]` generates code that calls the same trait on every field:

```rust
impl Clone for Block {
    fn clone(&self) -> Self {
        Block {
            header: self.header.clone(),            // requires BlockHeader: Clone
            transactions: self.transactions.clone(), // requires Vec<Transaction>: Clone
        }
    }
}
```

So for `Block: Clone` to compile, every field must also be `Clone`. The requirement chains
down recursively:

```
Block: Clone
  → BlockHeader: Clone
  → Vec<Transaction>: Clone
      → Transaction: Clone
```

The same applies to every derived trait:

| Trait | What it does to each field |
|---|---|
| `Clone`       | calls `.clone()` on each field |
| `Debug`       | calls `{:?}` formatting on each field |
| `Serialize`   | serializes each field |
| `Deserialize` | deserializes each field |

`Vec<T>` is already covered by the standard library — it implements `Clone`, `Debug`,
`Serialize` etc. as long as `T` does. So you only need to derive the traits on your own
types (`BlockHeader`, `Transaction`, etc.) and the chain satisfies itself.

If any type in the chain is missing a trait the compiler tells you exactly where it broke.
This is the same error seen earlier when `U256` didn't implement `Serialize` —
`Hash(U256)` couldn't be `Serialize` because its field wasn't.
