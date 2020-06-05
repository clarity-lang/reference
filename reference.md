# Clarity

This file contains the reference for the Clarity language. 

## Language design

Clarity differs from most other smart contract languages in two essential ways:

* The language is interpreted and broadcasted on the blockchain as is (not compiled)
* The language is decidable (not Turing complete)
  
Using an interpreted language ensures that the executed code is
human-readable and auditable. A decidable language like Clarity makes
it possible to determine precisely which code is going to be executed,
for any function.

A Clarity smart contract is composed of two parts-- a data space
and a set of functions. Only the associated smart contract may modify
its corresponding data space on the blockchain. Functions may be
private and thus callable only from within the smart contract, or
public and thus callable from other contracts. Users call smart
contracts' public functions by broadcasting a transaction on the
blockchain which invokes the public function. Contracts can also call
public functions from other smart contracts.

Note some of the key Clarity language rules and limitations.

* The only atomic types are booleans, integers, fixed length buffers,
  and principals
* Recursion is illegal and there is no lambda function.
* Looping may only be performed via `map`, `filter`, or `fold`
* There is support for lists of the atomic types, however, the only
  variable length lists in the language appear as function inputs;
  There is no support for list operations like append or join.
* Variables are created via `let` binding and there is no support for
  mutating functions like `set`.


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
