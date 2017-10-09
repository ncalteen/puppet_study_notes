# Thinking Declarative

- Imperative programming issues commands that change a target's state.
  - __Procedural Programming__: State changes are handled within procedures/functions to avoid duplication.

## Handling Change

- When you write code to perform a sequence of operations, the sequence may change the final state if it is run more than once.

  ```bash
  # This will error out on the second command.
  # It will also fail on some OSes.
  useradd -u 1001 -g 1001 -c "Joe" -m joe
  useradd -u 1001 -g 1001 -c "Joe" -m joe
  ```

## Using Idempotence

- The operation achieves the same results every time it executes.
- A configuration manifest can be applied/reapplied (converged) to always achieve the desired state.
- Idempotent code must be able to compare, evaluate, and apply every resource, as well as every attribute of every resource.

## Declaring Final State

- Configuration language must avoid describing the actions required to reach a desired state.
  - It should describe the desired state, and leave actions up to the interpreter.

  ```puppet
  user { 'joe':
    ensure     => present,
    uid        => '1001',
    gid        => '1001',
    comment    => 'Joe',
    managehome => true,
  }
  ```