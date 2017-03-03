# tla+ ∪ rust → <img src="parrot.gif" width="48" height="48" />

my goal: verify core lock-free and distributed algorithms in use
with [rsdb](http://github.com/spacejam/rsdb) and
[rasputin](http://github.com/disasters/rasputin).

- [x] [intro: processes and labels](#here-we-go-jumping-into-pluscal)
- [ ] [lock-free ring buffer](#lock-free-ring-buffer)

# here we go... jumping into pluscal

This is a summary of an example from
[a wonderful primer on TLA+](https://www.learntla.com/introduction/example/)...

The first thing to know is that there are two languages in play: pluscal and TLA.
We test code using `tlc`, which understands most of TLA (not infinite sets, maybe
other stuff). TLA started as a specification language, tlc came along later to
actually test it, and pluscal is a simpler language that can be transpiled into
TLA. Pluscal has two forms, `c` and `p`. They are functionally identical, but
`c` form uses braces and `p` form uses prolog/ruby-esque `begin` and `end`
statements that can be a little easier to spot errors with, in my opinion.

We're writing Pluscal in a TLA comment (block comments are written
with `(* <comment text> *)`), and when we run a translator like `pcal2tla`
it will insert TLA after the comment, in the same file.

```tla
------------------------------- MODULE pcal_intro -------------------------------
EXTENDS Naturals, TLC

(* --algorithm transfer
variables alice_account = 10, bob_account = 10,
          account_total = alice_account + bob_account

process TransProc \in 1..2
  variables money \in 1..20;
begin
  Transfer:
    if alice_account >= money then
      A: alice_account := alice_account - money;
      B: bob_account := bob_account + money;
    end if;
C: assert alice_account >= 0;
end process

end algorithm *)

\* this is a TLA comment. pcal2tla will insert the transpiled TLA here

MoneyInvariant == alice_account + bob_account = account_total

=============================================================================
```

This code specifies 3 global variables, `alice_account`, `bob_account`, `account_total`.
It specifies, using `process <name> \in 1..2` that it will run in two concurrent processes.
Each concurrent process has local state, `money`, which may take any initial value from
1 to 20, inclusive.  It defines steps `Transfer`, `A`, `B` and `C` which are evaluated as
atomic units, although they will be tested against all possible interleavings of execution.
All possible values will be tested.

Let's save the above example as `pcal_intro.tla`, transpile the pluscal comment to TLA,
then run it with tlc! (if you want to name it something else, update the MODULE
specification at the top)

```
pcal2tla pcal_intro.tla
tlc pcal_intro.tla
```

BOOM! This blows up because our transaction code sucks, big time. Looking at the trace
that tlc outputs, it shows us how alice's account may become negative. Because processes
1 and 2 execute the steps sequentially but with different interleavings, the algorithm
will check `alice_account >= money` before trying to transfer it to bob. By the time one
process subtracts the code from alice, however, the other process may have already done so.
We can specify that these steps and checks happen atomically by changing:

```
  Transfer:
    if alice_account >= money then
      A: alice_account := alice_account - money;
      B: bob_account := bob_account + money;
    end if;
```

to

```
  Transfer:
    if alice_account >= money then
      \* remove the labels A: and B:
      alice_account := alice_account - money;
      bob_account := bob_account + money;
    end if;
```

which means that the entire `Transfer` step is atomic. In reality, maybe this is done
by punting this atomicity requirement to a database transaction. Re-running tlc should
produce no errors now, because both processes atomically check + deduct + add balances
to the bank accounts without violating the assertion.

The invariants at the bottom are not actually being checked yet. They are specified in
TLA, not in the pluscal comment. They can be checked by creating a `pcal_intro.cfg` file
(or replace the one auto-generated by pcal2tla) with the following content:

```
SPECIFICATION Spec
INVARIANT MoneyInvariant
```

# lock-free ring buffer

Let's write a lock-free ring buffer!
