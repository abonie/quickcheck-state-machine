% State machine modelling and property based testing combined with fault injection
% Stevan Andjelkovic
% 2019.3.22, BOBKonf (Berlin)

---

# Motivation

* Fault tolerant (distributed) systems, hard to get right (many edge cases)

* *Simple Testing Can Prevent Most Critical Failures* paper [@yuan14]

    + The authors studied 198 randomly sampled user-reported failures from five
      distributed systems (Cassandra, HBase, HDFS, MapReduce, Redis)

    + "Almost all catastrophic failures (48 in total – 92%) are the result of
      incorrect handling of non-fatal errors explicitly signalled in software."

    + Example: `... } catch (Exception e) { LOG.error(e); // TODO: we should retry here! }`

---

# Related work

* Chaos engineering (Netflix)

* Jepsen (Kyle "aphyr" Kingsbury)

---

# Overview

* Property based testing (pure/side-effect free/stateless programs)
* State machine modelling (monadic/has side-effect/stateful programs)
* Fault injection (provoking exceptions)
* Using the
  [`quickcheck-state-machine`](https://github.com/advancedtelematic/quickcheck-state-machine)
  Haskell library, but the principles are general

---

# Recap: property based testing

* Unit tests

```haskell
      test :: Bool
      test = reverse (reverse [1,2,3]) == [1,2,3]

```

* Property based tests

```haskell
      prop :: [Int] -> Bool
      prop xs = reverse (reverse xs) == xs
```

. . .

* Proof by (structural) induction

      $\quad\;\forall xs(\textsf{reverse}(\textsf{reverse}(xs)) = xs)$

. . .

* Type theory

```haskell
      proof : forall xs -> reverse (reverse xs) == xs
```

---

# State machine modelling (somewhat simplified)

* Datatype of actions/commands that users can perform
* A simplified model of the system
* A transition function explaining how the model evolves for each action
* Semantics function that executes the action against the real system
* Post-condition that asserts that the result of execution matches the model

---

# Example: CRUD application

* ```haskell
    data Action = Create | Read | Update String | Delete
    ```
* ```haskell
    type Model = Maybe String
    ```
* ```haskell
    transition :: Model -> Action -> Model
```

* XXX:

[//]: ![State machine model](image/asm.jpg)\

---

# The `quickcheck-state-machine` library

* Use abstract state machine to model the program
    - A model datatype, and an initial model
    - A datatype of actions (things that can be happen in the system we are modelling)
    - A transition function that given an action advances the model to the
      next state

* A semantics function that takes an action and runs it against the real system

* Use pre- and post-conditions on the model to make sure that the model agrees
  with reality

* Use QuickCheck's generation to conduct experiments that validate the
  model

---

# Fault injection

* Many different tools and libraries, none native to Haskell
* We'll use the C library `libfiu` (**f**ault **i**njection in **u**serspace)
* Two modes of operation
    + Inject POSIX API/syscall failures
    + Inject failures at user specified failpoints

---

# Fault injection: syscall failures

* Using `fiu-run` directly:

```bash
      fiu-run -x -c 'enable name=posix/io/*' ls
```

* Via `fiu-ctrl` in a possibly different process:

```bash
      fiu-run -x top

      fiu-ctrl -c "enable  name=posix/io/oc/open" \
          $(pidof top)
      fiu-ctrl -c "disable name=posix/io/oc/open" \
          $(pidof top)
```

---

# Fault injection: user specified failpoints


```c
size_t free_space() {
        fiu_return_on("no_free_space", 0);

        [code to find out how much free space there is]
        return space;
}

bool file_fits(FILE *fd) {
        if (free_space() < file_size(fd)) {
                return false;
        }
        return true;
}
```

```c
fiu_init();
fiu_enable("no_free_space", 1, NULL, 0);
assert(file_fits("tmpfile") == false);
```

---

# Demo, "toy" example

---

# "Real world" examples

* libraft
* Cardano wallet

---

# Further work

* Fault injection library for Haskell, c.f. FreeBSD's failpoints and the Rust
  library `pingcap/fail-rs`

* [Jepsen](https://jepsen.io/)-like tests: parallel state machine testing with
  fault injection and linearisability

---

# Conclusion

---

# References
