# Gretchen

![Gretchen Wieners. She knows every transaction's business. She thinks about
every ordering of every history. That's why her state space is so big--it's
full of secrets.](doc/gretchen.gif)

Gretchen takes a history: a collection of transactions of reads or writes over
a set of named registers, and determines whether that history is
*serializable*: informally, whether there exists some sequential execution of
those transactions with equivalent effects. We adapt [Cerone, Bernardi, and
Gotsman's formulation of transactional consistency
models](http://drops.dagstuhl.de/opus/volltexte/2015/5375/pdf/15.pdf),
decomposing serializability into internal and external consistency, plus a
total visibility order, and verify external consistency by solving a constraint
problem for that ordering. We use fzn-gecode as our solver.

This problem is almost too NP to function, but with some heuristics, we should
be able to assess reasonably-sized histories.

Like [Knossos](https://github.com/jepsen-io/knossos), Gretchen is intended to
analyze observed histories of operations from real databases, generated by the
[Jepsen](https://github.com/jepsen-io/jepsen) distributed systems testing tool.

Gretchen is experimental. The input and output formats and error types will
probably change. It does detect known serializability errors, and finds
solutions for generated serializable histories. However, we have not verified
its behavior in depth, the solver often segfaults, and key optimizations are
not yet in place.

There is no support for verifying strict serializability, though we should be
able to add this by introducing restrictions on transaction order based on
invocation/completion order. There is no representation of predicates, so we
cannot verify phantoms. There is no support for indeterminate transactions,
which may have succeeded or failed; the caller is responsible for determining
the total set of transactions which constitute a history.

See `gretchen.core` for an in-depth discussion.

## Usage

You'll need the Flatzinc gecode constraint solver on your path. On Debian, you
can install it with `sudo apt-get install flatzinc`.

An operation is a map of the form:

```clj
{:f       A function, either :read or :write
 :k       A key, e.g. a keyword or string
 :v       A value to read or write}
```

A transaction is a map with a single mandatory key: a sequence of operations.
Gretchen may internally augment a transaction with additional fields.

```clj
{:ops [op1, op2, ...]}
```

As a shorthand, we can use `(t op1 op2 ...)` to build a transaction, `(r key
value)` for a read, and `(w key value)` for a write.

```clj
user=> (require '[gretchen.gen :refer [t r w]])
user=> (t (w :x 0) (r :y 1))
{:ops (0 {:f :read :k :y :v 1})}
```

A history is a map with an initial state, and a collection of transactions. For instance,

```clj
user=> (require '[gretchen.gen :as gen])
user=> (gen/history 2 {:x 0 :y 0})
{:initial {:epoch 0 :x 0 :y 0}
 :txns ({:ops [{:f :read :k :epoch :v 0}
               {:f :read :k :y :v 0}
               {:f :write :k :x :v 20}]}
        {:ops [{:f :read :k :epoch :v 0}
               {:f :read :k :x :v 20}
               {:f :read :k :x :v 20}
               {:f :write :k :x :v 58}
               {:f :read :k :y :v 0}]})}
```

To check a history, use `gretchen.core/check` with a solver. `check` returns an
augmented history, numbering transactions with an index `:i`, computing indices
of external reads and writes over values of registers, and deriving a
serialization of the history, if one exists. Gretchen introduces an initial transaction at `:i 0` to establish the initial state.

```clj
user=> (require '[gretchen.constraint.flatzinc :refer [flatzinc]])
user=> (require '[gretchen.core :as g])
user=> (g/check (gen/history 3 {:x 0 :y 0}) (flatzinc))
{:ext-reads {:epoch {0 (3 2 1)} :x {0 (2 1)} :y {0 (1) 36 (3 2)}}
 :ext-writes {:epoch {0 (0)} :x {0 (0)} :y {0 (0) 36 (1)}}
 :initial {:epoch 0 :x 0 :y 0}
 :solution ({:i 0
             :ops ({:f :write :k :x :v 0}
                   {:f :write :k :y :v 0}
                   {:f :write :k :epoch :v 0})}
            {:i 1
             :ops [{:f :read :k :epoch :v 0}
                   {:f :read :k :y :v 0}
                   {:f :read :k :x :v 0}
                   {:f :write :k :y :v 36}]}
            {:i 2
             :ops [{:f :read :k :epoch :v 0}
                   {:f :read :k :x :v 0}
                   {:f :read :k :x :v 0}
                   {:f :read :k :y :v 36}]}
            {:i 3 :ops [{:f :read :k :epoch :v 0} {:f :read :k :y :v 36}]})
 :txns ({:i 0
         :ops ({:f :write :k :x :v 0}
               {:f :write :k :y :v 0}
               {:f :write :k :epoch :v 0})}
        {:i 1
         :ops [{:f :read :k :epoch :v 0}
               {:f :read :k :y :v 0}
               {:f :read :k :x :v 0}
               {:f :write :k :y :v 36}]}
        {:i 2
         :ops [{:f :read :k :epoch :v 0}
               {:f :read :k :x :v 0}
               {:f :read :k :x :v 0}
               {:f :read :k :y :v 36}]}
        {:i 3 :ops [{:f :read :k :epoch :v 0} {:f :read :k :y :v 36}]})}
```

Gretchen can identify internal consistency errors; for instance, failing to
read a prior write from a transaction:

```clj
user=> (g/check {:txns [(t (w :x 1) (r :x 2))]}
                (flatzinc))
{:errors ({:error {:expected 1 :op {:f :read :k :x :v 2} :type :internal}
           :i 1
           :ops [{:f :write :k :x :v 1} {:f :read :k :x :v 2}]})
 ...}
```

Or external consistency errors. For instance, a lost update, where two
transactions both compare-and-set 0 to 1:

```clj
user=> (g/check {:initial {:x 0}
                 :txns [(t (r :x 0) (w :x 1))
                        (t (r :x 0) (w :x 1))]}
                (flatzinc))
{:error {:type :no-ext-solution}
  :ext-reads {:x {0 (2 1)}}
  :ext-writes {:x {0 (0) 1 (2 1)}}
  :initial {:x 0}
  :txns ({:i 0 :ops ({:f :write :k :x :v 0})}
         {:i 1 :ops ({:f :read :k :x :v 0} {:f :write :k :x :v 1})}
         {:i 2 :ops ({:f :read :k :x :v 0} {:f :write :k :x :v 1})})}
```

Or read skew, in which a transaction overwrites values between another
transaction's reads.

```clj
user=> (g/check {:initial {:x 0, :y 0}
                 :txns [(t (r :x 0) (r :y 1))
                        (t (w :x 1) (w :y 1))]}
                (flatzinc))
{:error {:type :no-ext-solution}
 :ext-reads {:x {0 (1)} :y {1 (1)}}
 :ext-writes {:x {0 (0) 1 (2)} :y {0 (0) 1 (2)}}
 :initial {:x 0 :y 0}
 :txns ({:i 0 :ops ({:f :write :k :x :v 0} {:f :write :k :y :v 0})}
        {:i 1 :ops ({:f :read :k :x :v 0} {:f :read :k :y :v 1})}
        {:i 2 :ops ({:f :write :k :x :v 1} {:f :write :k :y :v 1})})}
```

Or write skew. This history is legal under Snapshot Isolation, because the two
transactions' write sets do not intersect; however, either transaction's
completion would invalidate the other's reads.

```clj
user=> (g/check {:initial {:x 0, :y 0}
                 :txns [(t (r :x 0) (r :y 0) (w :x 1))
                        (t (r :x 0) (r :y 0) (w :y 2))]}
                (flatzinc))
{:error {:type :no-ext-solution}
 :ext-reads {:x {0 (2 1)} :y {0 (2 1)}}
 :ext-writes {:x {0 (0) 1 (1)} :y {0 (0) 2 (2)}}
 :initial {:x 0 :y 0}
 :txns ({:i 0 :ops ({:f :write :k :x :v 0} {:f :write :k :y :v 0})}
        {:i 1
         :ops ({:f :read :k :x :v 0}
               {:f :read :k :y :v 0}
               {:f :write :k :x :v 1})}
        {:i 2
         :ops ({:f :read :k :x :v 0}
               {:f :read :k :y :v 0}
               {:f :write :k :y :v 2})})}
```

For more examples, see `gretchen.core-test`.

## License

Copyright © 2016 Kyle Kingsbury

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
