# The privacy model of Secret Contracts

- [The privacy model of Secret Contracts](#the-privacy-model-of-secret-contracts)
  - [Init](#init)
  - [Handle](#handle)
  - [Query](#query)
  - [Inputs](#inputs)
    - [Encrypted](#encrypted)
    - [Not encrypted](#not-encrypted)
    - [What inputs can be trusted](#what-inputs-can-be-trusted)
    - [What inputs cannot be trusted](#what-inputs-cannot-be-trusted)
  - [State](#state)
  - [External query](#external-query)
    - [Encrypted](#encrypted-1)
    - [Not encrypted](#not-encrypted-1)
  - [Outputs](#outputs)
    - [Encrypted](#encrypted-2)
    - [Not encrypted](#not-encrypted-2)
  - [Data leakage attacks by detecting patterns in contract usage](#data-leakage-attacks-by-detecting-patterns-in-contract-usage)
    - [Differences in input sizes](#differences-in-input-sizes)
    - [Differences in state key sizes](#differences-in-state-key-sizes)
    - [Differences in value sizes on state read](#differences-in-value-sizes-on-state-read)
    - [Differences in key/value sizes on state write](#differences-in-keyvalue-sizes-on-state-write)
    - [Differences in state access order](#differences-in-state-access-order)
    - [Differences in output return values size](#differences-in-output-return-values-size)
    - [Differences in the amounts of output messages/callbacks](#differences-in-the-amounts-of-output-messagescallbacks)
    - [Differences in sizes of output messages/callbacks](#differences-in-sizes-of-output-messagescallbacks)
    - [Differences in the orders of output messages/callbacks](#differences-in-the-orders-of-output-messagescallbacks)
    - [Differences in the amounts of output logs/events](#differences-in-the-amounts-of-output-logsevents)
    - [Differences in sizes of output logs/events](#differences-in-sizes-of-output-logsevents)
    - [Differences in the orders of output logs/events](#differences-in-the-orders-of-output-logsevents)

## Init

## Handle

## Query

- Read-only access to state
- Metered by gas, but does not incurs fees
- Can be used to implement getters and decide what each user can see
- No msg.sender
- Access control: use passwords instead of checking for msg.sender
- Passwords can be generated for users via `Init` or `Handle`.

## Inputs

### Encrypted

- `msg` - Only known to the tx sender and the contract

### Not encrypted

- `msg.sender`
- funds sent

### What inputs can be trusted

- tx sender
- funds sent
- `msg`

### What inputs cannot be trusted

- `block.height`

## State

- Only known to that specific contract

## External query

### Encrypted

### Not encrypted

## Outputs

### Encrypted

- Only known to the tx sender and the contract

### Not encrypted

## Data leakage attacks by detecting patterns in contract usage

Depending on the contract's implementation, an attacker might be able to de-anonymization information about the contract and its clients. Contract developers need to consider all the following scenarios and more, and implement mitigation in case that some of these attack vectors can worsen the privacy aspect of their app.

In all the following scenarios, assume that an attacker has a local full node in its control. They cannot break into SGX, but they can tightly monitor and debug every other aspect of the node, including trying to feed old transactions directly to the contract inside SGX (replay). Also, though it's encrypted, they can also monitor memory (size), CPU (load) and disk usage (read/write timings and sizes) of the SGX chip.

For encryption, the Secret Network is using (AES-SIV)[https://tools.ietf.org/html/rfc5297], which does not pad the ciphertext. This means it leaks information about the plaintext data, specifically what is its size, though in most aspects it's more secure than other padded encryption schemes. Read more about the encryption specs [in here](protocol/encryption-specs.md).

### Differences in input sizes

An example input API for a contract with 2 `handle` functions:

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum HandleMsg {
    Send {
        amount: u8,
    },
    Transfer {
        amount: u8,
    },
}
```

This means that the inputs for txs on this contract would look like:

1. `{"send":{"amount":123}}`
2. `{"transfer":{"amount":123}}`

These inputs are encrypted, but by looking at their size an attacker can guess which function has been called by the user.

A quick fix for this issue might be renaming `Transfer` to `Tsfr`:

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum HandleMsg {
    Send {
        amount: u8,
    },
    Tsfr {
        amount: u8,
    },
}
```

Now an attacker wouldn't be able to tell which function was called:

1. `{"send":{"amount":123}}`
2. `{"tsfr":{"amount":123}}`

Although, if the attacker would know for example that `send.amount` is likely smaller than `100` and `tsfr.amount` is likely bigger than `100`, then they could still guess with some probability which function was called:

1. `{"send":{"amount":55}}`
2. `{"tsfr":{"amount":123}}`

Note that a client side solution can also be applied, but this is considered a very bad practice in infosec, as you cannot guarantee control of the client. E.g. you could pad the input to the maximum possible in this contract before encrypting it on the client side:

1. `{"send":{ "amount" : 55 } }`
2. `{"transfer":{"amount":123}}`

Again, this is very not recommended as you cannot guarantee control of the client!

### Differences in state key sizes

Contracts' state is stored on-chain inside a key-value store, thus the `key` must remain constant between calls. This means that if a contract uses storage keys with different sizes, an attacker might find out information about the execution of a contract.

Let's see an example for a contract with 2 `handle` functions:

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum HandleMsg {
    Send { amount: u8 },
    Tsfr { amount: u8 },
}

pub fn handle<S: Storage, A: Api, Q: Querier>(
    deps: &mut Extern<S, A, Q>,
    _env: Env,
    msg: HandleMsg,
) -> HandleResult {
    match msg {
        HandleMsg::Send { amount } => {
            deps.storage.set(b"send", &amount.to_be_bytes());
            Ok(HandleResponse::default())
        }
        HandleMsg::Tsfr { amount } => {
            deps.storage.set(b"transfer", &amount.to_be_bytes());
            Ok(HandleResponse::default())
        }
    }
}
```

By looking at state write operation, an attacker can guess which function was called based on the size of the key that was used to write to storage:

1. `send`
2. `transfer`

Again, some quick fixes for this issue might be:

1. Renaming `transfer` to `tsfr`.
2. Padding `send` to have the same length as `transfer`: `sendsend`.

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum HandleMsg {
    Send { amount: u8 },
    Tsfr { amount: u8 },
}

pub fn handle<S: Storage, A: Api, Q: Querier>(
    deps: &mut Extern<S, A, Q>,
    _env: Env,
    msg: HandleMsg,
) -> HandleResult {
    match msg {
        HandleMsg::Send { amount } => {
            deps.storage.set(b"sendsend", &amount.to_be_bytes());
            Ok(HandleResponse::default())
        }
        HandleMsg::Tsfr { amount } => {
            deps.storage.set(b"transfer", &amount.to_be_bytes());
            Ok(HandleResponse::default())
        }
    }
}
```

### Differences in value sizes on state read

### Differences in key/value sizes on state write

### Differences in state access order

### Differences in output return values size

### Differences in the amounts of output messages/callbacks

### Differences in sizes of output messages/callbacks

### Differences in the orders of output messages/callbacks

### Differences in the amounts of output logs/events

### Differences in sizes of output logs/events

### Differences in the orders of output logs/events