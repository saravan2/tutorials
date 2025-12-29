# Rust Tutorial: Patterns Used in Cloud Hypervisor

This tutorial covers the Rust language features and patterns used in implementing the ACPI Generic Initiator feature for Cloud Hypervisor.

## Table of Contents
1. [Structs and Data Structures](#structs-and-data-structures)
2. [Enums and Pattern Matching](#enums-and-pattern-matching)
3. [Traits and Implementations](#traits-and-implementations)
4. [Error Handling](#error-handling)
5. [Option and Result Types](#option-and-result-types)
6. [Collections](#collections)
7. [Attributes and Macros](#attributes-and-macros)
8. [Memory Representation](#memory-representation)
9. [String Handling](#string-handling)
10. [References and Lifetimes](#references-and-lifetimes)

---

## 1. Structs and Data Structures

### Basic Struct Definition

```rust
#[derive(Clone, Debug, PartialEq, Eq, Deserialize, Serialize)]
pub struct GenericInitiatorConfig {
    pub pci_bdf: String,
    pub numa_node: u32,
}
```

**Breaking it down:**

- `#[derive(...)]` - Automatically implements common traits
  - `Clone` - Allows creating copies with `.clone()`
  - `Debug` - Enables `{:?}` formatting for debugging
  - `PartialEq, Eq` - Enables equality comparison (`==`, `!=`)
  - `Deserialize, Serialize` - Enables JSON/YAML serialization (from `serde` crate)

- `pub` - Makes the struct and its fields public
- `String` - Owned, heap-allocated string
- `u32` - 32-bit unsigned integer

### Struct with Default Values

```rust
#[derive(Default, IntoBytes, Immutable, FromBytes)]
struct GenericInitiatorAffinity {
    pub type_: u8,
    pub length: u8,
    _reserved1: u16,
    pub device_handle_type: u8,
    pub proximity_domain: u32,
    pub device_handle: [u8; 16],
    pub flags: u32,
    _reserved2: u32,
}
```

**Key points:**

- `_reserved1`, `_reserved2` - Leading underscore suppresses "unused field" warnings
- `[u8; 16]` - Fixed-size array of 16 bytes
- `Default` trait - Allows `GenericInitiatorAffinity::default()`
- `IntoBytes, FromBytes` - Traits for binary serialization

### Packed Structs (C-compatible layout)

```rust
#[repr(C, packed)]
struct GenericInitiatorAffinity {
    // fields...
}
```

- `#[repr(C, packed)]` - Uses C memory layout with no padding
- Critical for ACPI structures that must match exact binary specifications
- Size matters: `std::mem::size_of::<GenericInitiatorAffinity>()` must be exactly 32 bytes

---

## 2. Enums and Pattern Matching

### Enum Definition

```rust
#[derive(Debug, PartialEq, Eq, Error)]
pub enum ValidationError {
    InvalidGenericInitiatorBdf(String),
    InvalidGenericInitiatorNode(u32),
    GenericInitiatorBdfNotVfio(String),
    GenericInitiatorDuplicateBdf(String),
}
```

**Key concepts:**

- Enums can hold data in each variant
- `Error` trait (from `thiserror` crate) makes this usable as an error type
- Variants can contain different types: `String`, `u32`, etc.

### Pattern Matching with `match`

```rust
match self {
    InvalidGenericInitiatorBdf(s) => {
        write!(f, "Invalid BDF format for generic initiator: {s}")
    }
    InvalidGenericInitiatorNode(n) => {
        write!(f, "Generic initiator references non-existent NUMA node: {n}")
    }
    GenericInitiatorBdfNotVfio(s) => {
        write!(f, "Generic initiator BDF {s} does not refer to a VFIO device")
    }
    GenericInitiatorDuplicateBdf(s) => {
        write!(f, "Duplicate BDF in generic initiator configuration: {s}")
    }
}
```

**Pattern matching features:**

- Destructures enum variants to extract inner values
- Must be exhaustive (handle all cases)
- Can use `_` for catch-all cases
- Variables in patterns (like `s`, `n`) bind to the inner values

### `if let` Pattern Matching

```rust
if let Some(gi_list) = generic_initiators {
    // gi_list is unwrapped and available here
    for gi_config in gi_list {
        // process each config
    }
}
```

**When to use:**

- When you only care about one pattern
- Cleaner than `match` with a single arm
- Common with `Option` and `Result` types

---

## 3. Traits and Implementations

### Associated Functions (Static Methods)

```rust
impl GenericInitiatorConfig {
    pub const SYNTAX: &'static str =
        "Generic Initiator parameters \"pci_bdf=<segment:bus:device.function>,node=<node_id>\"";

    pub fn parse(generic_initiator: &str) -> Result<Self> {
        // implementation
        Ok(GenericInitiatorConfig {
            pci_bdf,
            numa_node,
        })
    }

    pub fn validate(&self, numa_nodes: &arch::NumaNodes, num_pci_segments: u16) -> ValidationResult<()> {
        // implementation
        Ok(())
    }
}
```

**Key concepts:**

- `impl StructName` - Implementation block for a struct
- `pub const` - Public constant associated with the type
- `&'static str` - String literal with static lifetime (lives for program duration)
- `fn parse(...) -> Result<Self>` - Constructor that can fail
- `Self` - Refers to the type being implemented (`GenericInitiatorConfig`)
- `&self` - Immutable reference to self (first parameter in methods)

### Custom Methods

```rust
impl GenericInitiatorAffinity {
    fn from_pci_bdf(bdf: PciBdf, proximity_domain: u32) -> Self {
        let mut device_handle = [0u8; 16];

        // Modify device_handle
        let segment = bdf.segment();
        device_handle[0] = (segment & 0xff) as u8;
        device_handle[1] = ((segment >> 8) & 0xff) as u8;

        Self {
            type_: 5,
            length: 32,
            _reserved1: 0,
            device_handle_type: 1,
            proximity_domain,
            device_handle,
            flags: 1,
            _reserved2: 0,
        }
    }
}
```

**Pattern: Constructor pattern**

- Takes parameters and constructs the struct
- `Self { ... }` - Struct initialization syntax
- Fields can be specified in any order
- Shorthand: `proximity_domain` is same as `proximity_domain: proximity_domain`

---

## 4. Error Handling

### Defining Custom Errors

```rust
#[derive(Debug, Error)]
pub enum Error {
    #[error("Error parsing --generic-initiator: {0}")]
    ParseGenericInitiator(#[source] OptionParserError),

    #[error("Error parsing --generic-initiator: pci_bdf field missing")]
    ParseGenericInitiatorBdfMissing,
}
```

**Using `thiserror` crate:**

- `#[error("...")]` - Defines the error message
- `{0}` - Formats the first field
- `#[source]` - Marks the underlying error cause
- Automatically implements `Display` and `Error` traits

### Propagating Errors with `?`

```rust
pub fn parse(generic_initiator: &str) -> Result<Self> {
    let mut parser = OptionParser::new();
    parser.add("pci_bdf").add("node");

    parser
        .parse(generic_initiator)
        .map_err(Error::ParseGenericInitiator)?;  // Convert error type

    let pci_bdf = parser
        .get("pci_bdf")
        .ok_or(Error::ParseGenericInitiatorBdfMissing)?;  // Convert Option to Result

    Ok(GenericInitiatorConfig { pci_bdf, numa_node })
}
```

**Error handling patterns:**

- `?` - If error, return early; if ok, unwrap the value
- `.map_err(|e| ...)` - Transform error type
- `.ok_or(error)` - Convert `Option<T>` to `Result<T, E>`

### Custom Result Types

```rust
type ValidationResult<T> = std::result::Result<T, ValidationError>;
pub type Result<T> = result::Result<T, Error>;
```

**Type aliases:**

- Makes code more concise
- `ValidationResult<()>` returns nothing on success, error on failure
- Common pattern in Rust libraries

---

## 5. Option and Result Types

### Option<T> - Nullable Values

```rust
pub generic_initiators: Option<Vec<GenericInitiatorConfig>>,
```

**Two variants:**

- `Some(value)` - Contains a value
- `None` - No value (like null in other languages)

**Common operations:**

```rust
// Checking if Some
if let Some(gi_list) = &vm_params.generic_initiators {
    // gi_list is available here
}

// Unwrapping with default
let gi = config.generic_initiators.unwrap_or(Vec::new());

// Mapping
let count = config.generic_initiators.map(|list| list.len());

// ok_or - Convert to Result
let gi_list = config.generic_initiators.ok_or(Error::Missing)?;
```

### Result<T, E> - Fallible Operations

```rust
pub fn validate(&self) -> ValidationResult<()> {
    if !self.is_valid() {
        return Err(ValidationError::Invalid);
    }
    Ok(())
}
```

**Two variants:**

- `Ok(value)` - Success
- `Err(error)` - Failure

**Usage patterns:**

```rust
// Pattern matching
match result {
    Ok(value) => println!("Success: {}", value),
    Err(e) => eprintln!("Error: {}", e),
}

// ? operator (early return on error)
let value = risky_operation()?;

// map - Transform Ok value
let result = parse_number("42").map(|n| n * 2);

// map_err - Transform Err value
let result = operation().map_err(|e| MyError::from(e))?;
```

---

## 6. Collections

### BTreeMap - Ordered Key-Value Storage

```rust
let numa_node_ids: BTreeSet<u32> = if let Some(numa) = &self.numa {
    numa.iter().map(|n| n.guest_numa_id).collect()
} else {
    BTreeSet::new()
};
```

**Features:**

- Sorted by key
- `.contains_key(&key)` - Check if key exists
- `.get(&key)` - Get value by key (returns `Option`)
- `.insert(key, value)` - Add or update
- `.iter()` - Iterator over `(&K, &V)` pairs

### BTreeSet - Ordered Set

```rust
let mut seen_bdfs = BTreeSet::new();
for gi in gi_list {
    if !seen_bdfs.insert(&gi.pci_bdf) {
        return Err(ValidationError::Duplicate);
    }
}
```

**Features:**

- No duplicates
- Sorted elements
- `.insert(value)` - Returns `false` if already present
- `.contains(&value)` - Check membership

### Vec - Dynamic Array

```rust
let mut gi_config_list = Vec::new();
for item in gi_list.iter() {
    let gi_config = GenericInitiatorConfig::parse(item)?;
    gi_config_list.push(gi_config);
}
```

**Common operations:**

- `.push(item)` - Add to end
- `.pop()` - Remove from end (returns `Option`)
- `.len()` - Number of elements
- `.iter()` - Immutable iterator
- `.iter_mut()` - Mutable iterator

### HashMap vs BTreeMap

```rust
use std::collections::HashMap;        // Unordered, faster
use std::collections::BTreeMap;       // Ordered, slightly slower
```

**When to use:**

- `HashMap` - When you don't care about order
- `BTreeMap` - When you need sorted keys (used in Cloud Hypervisor for consistency)

---

## 7. Attributes and Macros

### Derive Macros

```rust
#[derive(Clone, Debug, PartialEq, Eq, Deserialize, Serialize)]
```

**Common derives:**

- `Clone` - Enables `.clone()`
- `Copy` - Enables implicit copying (for small types)
- `Debug` - Enables `{:?}` formatting
- `PartialEq` - Enables `==`, `!=`
- `Eq` - Marker for full equality
- `Default` - Provides `::default()`
- `Serialize, Deserialize` - JSON/YAML support (serde)

### Conditional Compilation

```rust
#[cfg(target_arch = "x86_64")]
pub sgx_epc: Option<Vec<SgxEpcConfig>>,

#[cfg(target_arch = "aarch64")]
pub const ACPI_APIC_GENERIC_CPU_INTERFACE: u8 = 11;

#[cfg(feature = "tdx")]
let tdx_enabled = self.platform.as_ref().map(|p| p.tdx).unwrap_or(false);
```

**Conditional attributes:**

- `#[cfg(target_arch = "x86_64")]` - Only on x86_64
- `#[cfg(target_arch = "aarch64")]` - Only on ARM64
- `#[cfg(feature = "tdx")]` - Only if `tdx` feature enabled
- `#[cfg(test)]` - Only in test builds

### Other Useful Attributes

```rust
#[allow(dead_code)]           // Suppress unused code warning
struct Internal { }

#[repr(C, packed)]            // C memory layout, no padding
struct AcpiStructure { }

#[serde(default)]             // Use default if field missing in JSON
pub optional_field: bool,

#[serde(skip)]                // Don't serialize this field
pub internal_state: Vec<i32>,
```

---

## 8. Memory Representation

### Fixed-Size Arrays

```rust
pub device_handle: [u8; 16],   // Exactly 16 bytes
```

**Features:**

- Size known at compile time
- Allocated on stack (fast)
- `device_handle[0]` - Access by index
- `device_handle[2..5]` - Slice access

### Packed Structs for Binary Protocols

```rust
#[repr(C, packed)]
#[derive(Default, IntoBytes, Immutable, FromBytes)]
struct GenericInitiatorAffinity {
    pub type_: u8,                    // Offset 0
    pub length: u8,                   // Offset 1
    _reserved1: u16,                  // Offset 2
    pub device_handle_type: u8,       // Offset 4
    pub proximity_domain: u32,        // Offset 5 (no padding!)
    pub device_handle: [u8; 16],      // Offset 9
    pub flags: u32,                   // Offset 25
    _reserved2: u32,                  // Offset 29
}
```

**Why packed?**

- ACPI spec requires exact byte layout
- No padding between fields
- Must be 32 bytes total

### Size Assertions

```rust
assert_eq!(std::mem::size_of::<GenericInitiatorAffinity>(), 32);
assert_eq!(std::mem::size_of::<MemoryAffinity>(), 40);
```

**Purpose:**

- Verify struct size at runtime
- Catch alignment issues early
- Critical for binary protocols

### Bit Manipulation

```rust
// Extract bits from BDF
let segment = bdf.segment();
device_handle[0] = (segment & 0xff) as u8;           // Lower 8 bits
device_handle[1] = ((segment >> 8) & 0xff) as u8;    // Upper 8 bits

// Combine device and function
device_handle[3] = (bdf.device() << 3) | bdf.function();
```

**Operators:**

- `&` - Bitwise AND (masking)
- `|` - Bitwise OR (combining)
- `<<` - Left shift (multiply by power of 2)
- `>>` - Right shift (divide by power of 2)
- `as u8` - Type cast

---

## 9. String Handling

### String Types

```rust
pub pci_bdf: String,          // Owned, heap-allocated
pub fn parse(s: &str) -> {}   // Borrowed string slice
const SYNTAX: &'static str    // String literal
```

**Differences:**

- `String` - Owned, mutable, heap-allocated
- `&str` - Borrowed, immutable reference
- `&'static str` - String literal, lives for entire program

### String Operations

```rust
// Creating strings
let owned = "hello".to_string();
let owned2 = String::from("world");

// Concatenation
let combined = format!("{} {}", owned, owned2);

// Borrowing
fn process(s: &str) { }
process(&owned);              // Borrow String as &str

// Conversion
let bdf = parser.get("pci_bdf").ok_or(Error)?;
let pci_bdf = bdf.to_string();  // Convert &str to String
```

### String Parsing

```rust
use std::str::FromStr;

let bdf = PciBdf::from_str(&self.pci_bdf)
    .map_err(|_| ValidationError::InvalidBdf(self.pci_bdf.clone()))?;
```

**FromStr trait:**

- `T::from_str(s)` - Parse string into type `T`
- Returns `Result<T, ParseError>`
- Can fail with error

---

## 10. References and Lifetimes

### References

```rust
fn validate(&self, numa_nodes: &arch::NumaNodes) -> Result<()>
//         ^self                ^borrowed parameter
```

**Types of references:**

- `&T` - Immutable reference (shared)
- `&mut T` - Mutable reference (exclusive)

**Rules:**

1. Any number of immutable references OR one mutable reference
2. References must always be valid (no dangling references)

### Lifetimes in Structs

```rust
pub struct VmParams<'a> {
    pub cpus: &'a str,
    pub memory: &'a str,
    pub generic_initiators: Option<Vec<&'a str>>,
}
```

**Lifetime `'a`:**

- Links the lifetime of references to the struct
- `VmParams` cannot outlive the data it references
- Compiler enforces this at compile time

### Lifetime Elision

```rust
// Explicit lifetime
fn parse<'a>(input: &'a str) -> Result<&'a str>

// Elided (compiler infers)
fn parse(input: &str) -> Result<&str>
```

**Elision rules:**

1. Each parameter gets its own lifetime
2. If one input lifetime, it's assigned to all outputs
3. If `&self`, its lifetime is assigned to all outputs

### Borrowing Patterns

```rust
// Immutable borrow
let gi = &config.generic_initiators;

// Multiple immutable borrows OK
let gi1 = &config.generic_initiators;
let gi2 = &config.generic_initiators;

// Mutable borrow (exclusive)
let gi_mut = &mut config.generic_initiators;

// Can't have other borrows while mutable borrow exists
// let gi2 = &config.generic_initiators;  // ERROR!
```

---

## Practical Examples from the Implementation

### Example 1: Parsing with Error Handling

```rust
impl GenericInitiatorConfig {
    pub fn parse(generic_initiator: &str) -> Result<Self> {
        // Create parser
        let mut parser = OptionParser::new();
        parser.add("pci_bdf").add("node");

        // Parse input, convert error type
        parser
            .parse(generic_initiator)
            .map_err(Error::ParseGenericInitiator)?;

        // Extract required field, error if missing
        let pci_bdf = parser
            .get("pci_bdf")
            .ok_or(Error::ParseGenericInitiatorBdfMissing)?
            .to_string();

        // Extract and convert required field
        let numa_node = parser
            .convert::<u32>("node")
            .map_err(Error::ParseGenericInitiator)?
            .ok_or(Error::ParseGenericInitiatorNodeMissing)?;

        // Return success
        Ok(GenericInitiatorConfig {
            pci_bdf,
            numa_node,
        })
    }
}
```

### Example 2: Validation with Early Returns

```rust
pub fn validate(&self, numa_nodes: &arch::NumaNodes, num_pci_segments: u16) -> ValidationResult<()> {
    // Parse BDF, return error if invalid
    let bdf = pci::PciBdf::from_str(&self.pci_bdf)
        .map_err(|_| ValidationError::InvalidGenericInitiatorBdf(self.pci_bdf.clone()))?;

    // Validate range
    if bdf.segment() >= num_pci_segments {
        return Err(ValidationError::InvalidPciSegment(bdf.segment()));
    }

    // Check existence
    if !numa_nodes.contains_key(&self.numa_node) {
        return Err(ValidationError::InvalidGenericInitiatorNode(self.numa_node));
    }

    // All checks passed
    Ok(())
}
```

### Example 3: Iterator Chains

```rust
let numa_node_ids: BTreeSet<u32> = if let Some(numa) = &self.numa {
    numa.iter()                      // Create iterator
        .map(|n| n.guest_numa_id)    // Transform each element
        .collect()                   // Collect into BTreeSet
} else {
    BTreeSet::new()                  // Empty set
};
```

### Example 4: Working with Options

```rust
// Extract field from deeply nested Option
let generic_initiators = &device_manager
    .lock()                          // Get mutex guard
    .unwrap()                        // Unwrap Result
    .config                          // Access field
    .lock()                          // Get another mutex guard
    .unwrap()                        // Unwrap Result
    .generic_initiators;             // Access final field

// Process if Some
if let Some(gi_list) = generic_initiators {
    for gi_config in gi_list {
        // Process each config
    }
}
```

---

## Common Patterns Summary

### Constructor Pattern

```rust
impl MyStruct {
    pub fn new(param: Type) -> Self {
        Self { field: param }
    }

    pub fn from_data(data: &[u8]) -> Result<Self> {
        // Parse and construct, can fail
        Ok(Self { /* fields */ })
    }
}
```

### Builder Pattern

```rust
parser
    .add("field1")
    .add("field2")
    .parse(input)?;
```

### Error Propagation Chain

```rust
let result = operation1()?
    .operation2()?
    .operation3()?;
```

### Iterator Processing

```rust
collection
    .iter()
    .filter(|item| condition)
    .map(|item| transform(item))
    .collect()
```

---

## Additional Resources

- **The Rust Book**: https://doc.rust-lang.org/book/
- **Rust by Example**: https://doc.rust-lang.org/rust-by-example/
- **Rust Standard Library**: https://doc.rust-lang.org/std/
- **thiserror crate**: https://docs.rs/thiserror/
- **serde crate**: https://serde.rs/

## Next Steps

1. Practice with small examples of each pattern
2. Read through Cloud Hypervisor code with new understanding
3. Try modifying existing code
4. Write unit tests to understand behavior

---

*This tutorial covers the core Rust patterns used in systems programming projects like Cloud Hypervisor. Understanding these patterns will help you navigate and contribute to similar projects.*
