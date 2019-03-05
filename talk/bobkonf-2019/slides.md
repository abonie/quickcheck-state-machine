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

# Overview

* Property based testing (pure/side-effect free/stateless programs)
* State machine modelling (monadic/has side-effect/stateful programs)
* Fault injection (provoking exceptions)
* Examples using
    + [`quickcheck-state-machine`](https://github.com/advancedtelematic/quickcheck-state-machine)
      Haskell library for property based testing
    + The principles are general and tool independent

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

```haskell
data Action = Create | Read | Update String | Delete

type Model = Maybe String

transition :: Model -> Action -> Model
transition _m Create     = Just ""
transition  m Read       = m
transition _m (Update s) = Just s
transition _m Delete     = Nothing
```

---

# Example: CRUD application (continued)

```haskell
data Response = Unit () | String String

semantics :: Action -> IO Response
semantics Create     = Unit   <$> httpReq POST   url
semantics Read       = String <$> httpReq GET    url
semantics (Update s) = Unit   <$> httpReq PUT    url s
semantics Delete     = Unit   <$> httpReq DELETE url

postcondition :: Model -> Action -> Response -> Bool
postcondition (Just m) Read (String s) = s == m
postcondition _m       _act _resp      = True
```

---

# State machine modelling as a picture

![State machine model](../bobkonf-2018/image/asm.jpg)\

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

# Examples

* Over-the-air updates of cars (my workplace)
* Adjoint's
  [libraft](https://github.com/adjoint-io/raft/blob/master/test/QuickCheckStateMachine.hs)
* IOHK's blockchain [database](https://www.well-typed.com/blog/2019/01/qsm-in-depth/)

---

# Over-the-air updates (picture)

![OTA](image/ota.jpg)\


---

# Over-the-air updates

```haskell
data Action = Check | Download | Install | Reboot
            | Inject Fault | DisableFaultInject

data Fault = Network | Kill | GCIOPause | ProcessPrio
           | ReorderReq | SkewClock | RmFile | DamageFile
           | Libfiu (Either Syscall Failpoint)

inject :: Fault -> IO ()  -- Pseudo code
inject Network     = call "iptables -j DROP $IP"
inject Kill        = call "kill -9 $PID"
inject GCIOPause   = call "killall -s STOP $PID"
inject ProcessPrio = call "renice -p $PID"
inject ReorderReq  = call "someproxy" -- Which?
inject SkewClock   = call "faketime $SKEW"
inject RmFile      = call "rm -f $FILE"
inject DamageFile  = call "echo 0 > $FILE"
inject Libfiu      = call "fiu-ctrl -c $FAULT $PID"

```

---

# Over-the-air updates (continued)

```haskell
data Model = Model { fault :: Maybe Fault, ... }

transition :: Model -> Action -> Model
transition m (Fault f)          = m { fault = Just f }
transition m DisableFaultInject = m { fault = Nothing }
transition m ...

postcondition :: Model -> Action -> Response -> Bool
postcondition m act resp = case (fault m, act) of
  (Nothing,      Download) -> resp == Ok
  (Just Network, Download)
      -> resp == TimeoutError
  (Just (Libfiu (Right InstallFailure)), Install)
      -> resp == FailedToInstallError
  ...
```

---

# Over-the-air updates (continued 2)

```haskell
prop_reliable :: Property
prop_reliable = forAllActions $ \acts -> do
  (device, update) <- setup
  schedule update device
  let model = initModel { device = Just device }
  (model', result) <- runActions acts model
  assert (result == Success) -- Post-conditions hold
  runActions [ DisableFaultInject
             , Check, Download, Install, Reboot ]
             model'
  update' <- currentUpdate device
  assert (update' == update) -- Always able to recover
```

---

# Adjoint's `libraft`

* Adjoint's
  [libraft](https://github.com/adjoint-io/raft/blob/master/test/QuickCheckStateMachine.hs)
    + Simplified think of it as distributed and fault-tolerant "CRUD applicaiton
      example"
    + Injected faults: killing nodes and network traffic loss

---

# IOHK's blockchain database

* IOHK's blockchain database
    + File system mock tested against real file system
    + Database tests built on top of file system mock
    + Fault are injected into the file system mock

---

# Further work

* Fault injection library for Haskell, c.f.:
    + FreeBSD's [failpoints](https://www.freebsd.org/cgi/man.cgi?query=fail)
    + Rust's [`fail-rs`](https://github.com/pingcap/fail-rs) crate
    + Go's [`gofail`](https://github.com/etcd-io/gofail) library

* [Jepsen](https://jepsen.io/)-like tests: parallel state machine testing with
  fault injection and linearisability

* Lineage-driven fault injection [@alvaro15]

---

# Conclusion

* Fault injection can help causes exceptional circumstances

* Exceptional circumstances are by definition rare and hence less likely to be
  tested

* Exceptional circumstances are often edge cases and hence less likely to be
  considered when writing the program

* Exceptional circumstances will nevertheless occur in any long running system

* By combining fault injection with property based testing we force ourselves to
  consider these exceptional cases

---

# References
