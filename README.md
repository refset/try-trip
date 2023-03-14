# Try Trip: Datalog as a namespace

This webpage showcases [Trip](https://github.com/juxt/trip/)&mdash;a [single-namespace](https://github.com/juxt/trip/blob/main/src/juxt/trip/core.cljc) library for Clojure that compiles [DataScript](https://github.com/tonsky/datascript)-compatible Datalog into `for` loops&mdash;and is derived from Borkdude's excellent [cljs-showcase](https://borkdude.github.io/cljs-showcase) template.

The interactive snippets below are made possible using a combination of ClojureScript & [SCI](https://github.com/babashka/sci). SCI is required to support Trip's need for runtime access to `cljs.core/*eval*` (`eval` is used internally for query compilation).

The API documentation for Trip can be browsed [here](https://github.com/juxt/trip/blob/main/API.md).
<div style="width: 600px;" class="cljs-showcase">

``` clojure
(+ "Hello, " "ClojureScript REPL!")
```
</div>

Very basic error messages also show up as results:
<div style="width: 600px;" class="cljs-showcase">

``` clojure
(+ foo 1)
```
</div>

The Trip namespace is available to be require'd, so let's create our first database!
<div style="width: 600px;" class="cljs-showcase" data-cljs-showcase-no-editable="true">

``` clojure
(require '[juxt.trip.core :as trip])
(def t (trip/create-conn)) ;; an atom containing the result of `(trip/empty-db)`
```
</div>

A quick a look at the value of `t` shows that Trip's three internal indexes are empty:
<div style="width: 600px;" class="cljs-showcase">

``` clojure
@t ;; equivalent to `(trip/db t)`
```
</div>

Trip is completely schemaless, so can we simply insert a map containing a user-specified `db/id` value to create an entity in our database:
<div style="width: 600px;" class="cljs-showcase">

``` clojure
(trip/transact! t [{:db/id :foo :ref :bar}])
```
</div>

You can see that triples are printed as maps and the three indexes (`EAV`, `AEV`, `AVE`) have been populated ready for querying.

Now we can take a `db` value and query it with Datalog. Here we are asking for the set of all known IDs:
<div style="width: 600px;" class="cljs-showcase">

``` clojure
(->> (trip/db t)
     (trip/q '{:find [?e]
               :where [[?e :db/id]]}))
```
</div>

Explicit IDs underpin the dynamic and schemaless experience of using Trip, however this also means that updates happen at the granularity of entire entities. Transacting a new map for an existing entity will first retract all existing triples for that ID and then add the complete new set of triples (i.e. there is some cost of redundant work to the flexibility):
<div style="width: 600px;" class="cljs-showcase">

``` clojure
(-> (trip/transact! t [{:db/id :foo
                       :ref :bar
                       :val {:nested [:thing]}}])
    :tx-data)
```
</div>

If we insert an entity with an ID that happens to correlate with an existing value under an attribute, we can dynamically treat is as if it were a "reference" type attribute:
<div style="width: 600px;" class="cljs-showcase">

``` clojure
(trip/transact! t [{:db/id :bar :val "joined!"}])
(->> (trip/db t)
     (trip/q '{:find [?v]
               :where [[:foo :ref ?e2]
                       [?e2 :val ?v]]}))
```
</div>

This kind of dynamic behaviour is often referred to as "schema on read", which offers some benefits while prototyping&mdash;for instance it allows you to transact entities that refer to each other without worrying about dependency ordering&mdash;but it does mean you might want to consider enforcing referential integrity constraints through other means.

One way you can enforce constraints is by using Datalog to test for invariants during the `transact!` processing, which we can achieve by using Trip's transaction function capability:

<div style="width: 600px;" class="cljs-showcase">

``` clojure
;; reset the example data when evaluating the following snippet multiple times
(trip/transact! t [[:db.fn/retractEntity :baz] [:db.fn/retractEntity :foo2]])

(-> (trip/transact! t [{:db/id :baz :exists? true} ;; try commenting out this operation
                       [:db.fn/call (fn [db doc]
                                      (if (empty? (trip/q '{:find [?e]
                                                            :in [$ ?e]
                                                            :where [[?e :db/id]]}
                                                          db
                                                          (:ref doc)))
                                          [[:db.fn/cas :cancel-tx :cancel-tx :cancel-tx :cancel-tx]]
                                          [doc]))
                                    {:db/id :foo2 :ref :baz}]])
    :tx-data)
```
</div>

Should you wish to quickly compare Trip's API with DataScript, note that it is also available to be require'd throughout this webpage:

<div style="width: 600px;" class="cljs-showcase">

``` clojure
(require '[datascript.core :as d])
(d/empty-db)
```
</div>

That's a wrap! Thanks for taking a look at Trip and please give the [repo](https://github.com/juxt/trip/) a star. PRs, Issues and questions are welcome 🙏

<script src="js/main.js" type="application/javascript"></script>
