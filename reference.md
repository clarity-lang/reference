# Clarity

This file contains the reference for the Clarity language. 

## Design

A smart contract is composed of two parts:

1. A data-space, which is a set of tables of data which only the
   smart contract may modify
2. A set of functions which operate within the data-space of the
   smart contract, though they may call public functions from other smart
   contracts.

Users call smart contracts' public functions by broadcasting a
transaction on the blockchain which invokes the public function.

This smart contracting language differs from most other smart
contracting languages in two important ways:

1. The language _is not_ intended to be compiled. The LISP language
   described in this document is the specification for correctness.
2. The language _is not_ Turing complete. This allows us to guarantee
   that static analysis of programs to determine properties like
   runtime cost and data usage can complete successfully.

## Specifying Contracts

A smart contract definition is specified in a LISP language with the
following limitations:

1. Recursion is illegal and there is no `lambda` function.
2. Looping may only be performed via `map`, `filter`, or `fold`
3. The only atomic types are booleans, integers, fixed length
   buffers, and principals
4. There is additional support for lists of these types (and lists of
   lists), however the only variable length lists in the language
   appear as function inputs.
5. Variables may only be created via `let` binding and there
   is no support for mutating functions like `set`.
6. Defining of constants and functions are allowed for simplifying
   code using `define-private` statement. However, these are purely
   syntactic. If a definition cannot be inlined, the contract will be
   rejected as illegal. These definitions are also _private_, in that
   functions defined this way may only be called by other functions
   defined in the given smart contract.
7. Functions specified via `define-public` statements are _public_
   functions.
8. Functions specified via `define-read-only` statements are _public_
   functions and perform _no_ state mutations. Any attempts to 
   modify contract state by these functions or functions called by
   these functions will result in an error.

Public functions return a Response type result. If the function returns
an `ok` type, then the function call is considered valid, and any changes
made to the blockchain state will be materialized. If the function
returns an `err` type, it will be considered invalid, and will have _no
effect_ on the smart contract's state. So if function `foo.A` calls
`bar.B`, and `bar.B` returns an `ok`, but `foo.A` returns an `err`, no
effects from calling `foo.A` materialize--- including effects from
`bar.B`. If, however, `bar.B` returns an `err` and `foo.A` returns an `ok`,
there may be some database effects which are materialized from
`foo.A`, but _no_ effects from calling `bar.B` will materialize.

Unlike functions created by `define-public`, which may only return
Response types, functions created with `define-read-only` may return
any type.

## List Operations

* Lists may be multi-dimensional (i.e., lists may contain other lists), however each
  entry of this list must be of the same type.
* `filter` `map` and `fold` functions may only be called with user-defined functions
  (i.e., functions defined with `(define-private ...)`, `(define-read-only ...)`, or
  `(define-public ...)`) or simple native functions (e.g., `+`, `-`, `not`).
* Functions that return lists of a different size than the input size
  (e.g., `(append-item ...)`) take a required _constant_ parameter that indicates
  the maximum output size of the function. This is enforced with a runtime check.

## Data-Space Primitives

Data within a smart contract's data-space is stored within
`maps`. These stores relate a typed-tuple to another typed-tuple
(almost like a typed key-value store). As opposed to a table data
structure, a map will only associate a given key with exactly one
value. Values in a given mapping are set or fetched using:

1. `(map-get map-name key-tuple)` - This fetches the value
  associated with a given key in the map, or returns `none` if there
  is no such value.
2. `(map-set! map-name key-tuple value-tuple)` - This will set the
  value of `key-tuple` in the data map
3. `(map-insert! map-name key-tuple value-tuple)` - This will set
  the value of `key-tuple` in the data map if and only if an entry
  does not already exist.
4. `(map-delete! map-name key-tuple)` - This will delete `key-tuple`
   from the data map

We chose to use data maps as opposed to other data structures for two
reasons:

1. The simplicity of data maps allows for both a simple implementation
within the VM, and easier reasoning about functions. By inspecting a
given function definition, it is clear which maps will be modified and
even within those maps, which keys are affected by a given invocation.
2. The interface of data maps ensures that the return types of map
operations are _fixed length_, which is a requirement for static
analysis of smart contracts' runtime, costs, and other properties.

A smart contract defines the data schema of a data map with the
`define-map` call. The `define-map` function may only be called in the
top-level of the smart-contract (similar to `define-private`). This
function accepts a name for the map, and a definition of the structure
of the key and value types. Each of these is a list of `(name, type)`
pairs, and they specify the input and output type of `map-get`.
Types are either the values `'principal`, `'integer`, `'bool` or
the output of a call to `(buffer n)`, which defines an n-byte
fixed-length buffer. 

This interface, as described, disallows range-queries and
queries-by-prefix on data maps. Within a smart contract function,
you cannot iterate over an entire map.

## Clarity Type System

The Clarity language uses a strong static type system. Function arguments
and database schemas require specified types, and use of types is checked
during contract launch. The type system does _not_ have a universal
super type. The type system contains the following types:

* `(tuple (key-name-0 key-type-0) (key-name-1 key-type-1) ...)` -
  a typed tuple with named fields.
* `(list max-len entry-type)` - a list of maximum length `max-len`, with
  entries of type `entry-type`
* `(response ok-type err-type)` - object used by public functions to commit
  their changes or abort. May be returned or used by other functions as
  well, however, only public functions have the commit/abort behavior.
* `(optional some-type)` - an option type for objects that can either be
  `(some value)` or `none`
* `(buff max-len)` := byte buffer or maximum length `max-len`.
* `principal` := object representing a principal (whether a contract principal
  or standard principal).
* `bool` := boolean value (`true` or `false`)
* `int`  := signed 128-bit integer
* `uint` := unsigned 128-bit integer

### Type Admission

**UnknownType**. The Clarity type system does not allow for specifying
an "unknown" type, however, in type analysis, unknown types may be
constructed and used by the analyzer. Such unknown types are used
_only_ in the admission rules for `response` and `optional` types
(i.e., the variant types).

Type admission in Clarity follows the following rules:

* Types will only admit objects of the same type, i.e., lists will only
admit lists, tuples only admit tuples, bools only admit bools.
* A tuple type `A` admits another tuple type `B` iff they have the exact same
  key names, and every key type of `A` admits the corresponding key type of `B`.
* A list type `A` admits another list type `B` iff `A.max-len >= B.max-len` and
  `A.entry-type` admits `B.entry-type`.
* A buffer type `A` admits another buffer type `B` iff `A.max-len >= B.max-len`.
* An optional type `A` admits another optional type `B` if:
  * `A.some-type` admits `B.some-type` _OR_ `B.some-type` is an unknown type:
    this is the case if `B` only ever corresponds to `none`
* A response type `A` admits another response type `B` if one of the following is true:
  * `A.ok-type` admits `B.ok-type` _AND_ `A.err-type` admits `B.err-type`
  * `B.ok-type` is unknown _AND_ `A.err-type` admits `B.err-type`
  * `B.err-type` is unknown _AND_ `A.ok-type` admits `B.ok-type`
* Principals, bools, ints, and uints only admit types of the exact same type.

Type admission is used for determining whether an object is a legal argument for
a function, or for insertion into the database. Type admission is _also_ used
during type analysis to determine the return types of functions. In particular,
a function's return type is the least common supertype of each type returned from any
control path in the function. For example:

```
(define-private (if-types (input bool))
  (if input
     (ok 1)
     (err false)))
```

The return type of `if-types` is the least common supertype of `(ok
1)` and `(err false)` (i.e., the most restrictive type that contains
all returns). In this case, that type `(response int bool)`. Because
Clarity _does not_ have a universal supertype, it may be impossible to
determine such a type. In these cases, the functions are illegal, and
will be rejected during type analysis.

## Public Functions

Functions specified via `define-public` statements are _public_
functions and these are the only types of functions which may
be called directly through signed blockchain transactions. In addition
to being callable directly from a transaction (see the Stacks wire formats
for more details on Stacks transactions), public function may be called
by other smart contracts.

Public functions _must_ return a `(response ...)` type. This is used
by Clarity to determine whether or not to materialize any changes from
the execution of the function. If a function returns an `(err ...)`
type, and mutations on the blockchain state from executing the
function (and any function that it called during execution) will be
aborted.

In addition to function defined via `define-public`, contracts may expose
read-only functions. These functions, defined via `define-read-only`, are
callable by other smart contracts, and may be queryable via public blockchain
explorers. These functions _may not_ mutate any blockchain state. Unlike normal
public functions, read-only functions may return any type.

## Contract Calls

A smart contract may call functions from other smart contracts using a
`(contract-call?)` function.

This function returns a response type result-- the return value of the
called smart contract function.

We distinguish 2 different types of `contract-call?`:

* Static dispatch: the callee is a known, invariant contract available
on-chain when the caller contract is deployed. In this case, the
callee's principal is provided as the first argument, followed by the
name of the method and its arguments:

```scheme
(contract-call?
    .registrar
    register-name
    name-to-register)
```

* Dynamic dispatch: the callee is passed as an argument, and typed
as a trait reference (<A>).

```scheme
(define-public (swap (token-a <can-transfer-tokens>)
                     (amount-a uint)
                     (owner-a principal)
                     (token-b <can-transfer-tokens>)
                     (amount-b uint)
                     (owner-b principal)))
     (begin
         (unwrap! (contract-call? token-a transfer-from? owner-a owner-b amount-a))
         (unwrap! (contract-call? token-b transfer-from? owner-b owner-a amount-b))))
```

Traits can either be locally defined:

```scheme
(define-trait can-transfer-tokens (
    (transfer-from? (principal principal uint) (response uint)))
```

Or imported from an existing contract:

```scheme
(use-trait can-transfer-tokens
    .contract-defining-trait.can-transfer-tokens)
```

Looking at trait conformance, callee contracts have two different paths.
They can either be "compatible" with a trait by defining methods
matching some of the methods defined in a trait, or explicitely declare
conformance using the `impl-trait` statement:

```scheme
(impl-trait .contract-defining-trait.can-transfer-tokens)
```

Explicit conformance should be prefered when adequate.
It acts as a safeguard by helping the static analysis system to detect
deviations in method signatures before contract deployment.

The following limitations are imposed on contract calls:

1. On static dispatches, callee smart contracts _must_ exist at the
   time of creation.
2. No cycles may exist in the call graph of a smart contract. This
   prevents recursion (and re-entrancy bugs). Such structures can
   be detected with static analysis of the call graph, and will be
   rejected by the network.
3. `contract-call?` are for inter-contract calls only. Attempts to
   execute when the caller is also the callee will abort the
   transaction.

## Principals and tx-sender

Assets in the smart contracting language and blockchain are
"owned" by objects of the principal type, meaning that any object of
the principal type may own an asset. For the case of public-key hash
and multi-signature Stacks addresses, a given principal can operate on
their assets by issuing a signed transaction on the blockchain. _Smart
contracts_ may also be principals (reprepresented by the smart
contract's identifier), however, there is no private key associated
with the smart contract, and it cannot broadcast a signed transaction
on the blockchain.

A Clarity contract can use a globally defined `tx-sender` variable to
obtain the current principal. The following example defines a transaction
type that transfers `amount` microSTX from the sender to a recipient if amount
is a multiple of 10, otherwise returning a 400 error code.

```scheme
(define-public (transfer-to-recipient! (recipient principal) (amount uint))
  (if (is-eq (mod amount 10) 0)
      (stx-transfer? amount tx-sender recipient)
      (err u400)))
```

Clarity provides an additional variable to help smart contracts
authenticate a transaction sender. The keyword `contract-caller`
returns the principal that _called_ the current contract. If an
inter-contract call occurred, `contract-caller` returns the last
contract in the stack of callers. For example, suppose there are three
contracts A, B, and C, each with an `invoke` function such that
`A::invoke` calls `B::invoke` and `B::invoke` calls `C::invoke`.

When a user Bob issues a transaction that calls `A::invoke`, the value
of `contract-caller` in each successive invoke function's body would change:

```
in A::invoke,  contract-caller = Bob
in B::invoke,  contract-caller = A
in C::invoke,  contract-caller = B
```

This allows contracts to make assertions and perform authorization
checks using not only the `tx-sender` (which in this example, would
always be "Bob"), but also using the `contract-caller`. This could be
used to ensure that a particular function is only ever called directly
and never called via an inter-contract call (by asserting that
`tx-sender` and `contract-caller` are equal). We provide an example of
a two different types of authorization checks in the rocket ship
example below.

## Smart contracts as principals

Smart contracts themselves are principals and are represented by the
smart contract's identifier -- which is the publishing address of the
contract _and_ the contract's name, e.g.:

```scheme
'SZ2J6ZY48GV1EZ5V2V5RB9MP66SW86PYKKQ9H6DPR.contract-name
```

For convenience, smart contracts may write a contract's identifier in the
form `.contract-name`. This will be expanded by the Clarity interpreter into
a fully-qualified contract identifier that corresponds to the same
publishing address as the contract it appears in. For example, if the
same publisher key, `SZ2J6ZY48GV1EZ5V2V5RB9MP66SW86PYKKQ9H6DPR`, is 
publishing two contracts, `contract-A` and `contract-B`, the fully
qualified identifier for the contracts would be:

```scheme
'SZ2J6ZY48GV1EZ5V2V5RB9MP66SW86PYKKQ9H6DPR.contract-A
'SZ2J6ZY48GV1EZ5V2V5RB9MP66SW86PYKKQ9H6DPR.contract-B
```

But, in the contract source code, if the developer wishes
to call a function from `contract-A` in `contract-B`, they can
write

```scheme
(contract-call? .contract-A public-function-foo)
```

This allows the smart contract developer to modularize their
applications across multiple smart contracts _without_ knowing
the publishing key a priori.

In order for a smart contract to operate on assets it owns, smart contracts
may use the special `(as-contract ...)` function. This function
executes the expression (passed as an argument) with the `tx-sender`
set to the contract's principal, rather than the current sender. The
`as-contract` function returns the value of the provided expression.
