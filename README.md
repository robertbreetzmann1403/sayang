# sayang

*sayang* complects the definition of a function with its specification.

## Rationale

A useful summary of any function is the combination of its name, its expectations about the input and guarantees of its output.
[clojure.spec](https://clojure.org/about/spec) provides this kind of function summary with the [s/fdef](https://clojure.org/guides/spec#_spec_ing_functions) macro.
With *clojure.spec*, definition and specification of a function are separated concerns and there are good reasons for that decision. To quote [Alex Miller](https://groups.google.com/forum/m/#!topic/clojure/0wqvG2sef8I):
 > "There's a lot of value in separating the specs from the functions. You can put them in different places and only use just the specs (to spec an API for example) or just the functions (for production use where you don't need the specs)."

Still, when you try to read or change a function, it is helpful to have the specification right next to the implementation.
Additionally, if you need to change the number or the order of the function parameters, you only have to change it once.
That is why you can optionally complect definition and specification of a function with *sayang*.

### Syntax

While *sayang* values a resemblance to the syntax of [prismatic/schema](https://github.com/plumatic/schema), it values not interfering with the functionality of [Cursive](https://cursive-ide.com/) more.
Use the *resolve macro as*-Feature of *Cursive* to get the same tooling as function definitions with *clojure.core*.

### Alternatives

If you do not like the syntax or any other design decisions of *sayang*, you might want to have a look at the `defn-spec` macro of [Orchestra](https://github.com/jeaye/orchestra).

## Leiningen

*sayang* is available from Clojars. Add the following dependency to your *project.clj*:

[![Current Version](https://clojars.org/me.lomin/sayang/latest-version.svg)](https://clojars.org/me.lomin/sayang)

Generation of specifications is turned off by default. One option to activate it is by adding jvm-opts in Leiningen:

```clojure
(defproject sayang-test "0.1.0-SNAPSHOT"
  :dependencies [[org.clojure/clojure "1.9.0"]
                 [orchestra  "2017.11.12-1"]
                 [org.clojure/test.check  "0.9.0"]
                 [org.clojure/spec.alpha "0.1.143"]
                 [me.lomin/sayang "0.2.0"]]
  :profiles {:dev {:jvm-opts ["-Dme.lomin.sayang.activate=true"]}})
```

Alternatively, you can switch the activation toggle manually in your code:

```clojure
(alter-var-root #'me.lomin.sayang/*activate*  (constantly true))
```

Once activated, you need a runtime dependency to org.clojure/test.check.

## Features

* [in place function specification at definition](#specification-at-definition)
* [partial specification](#partial-specification)
* [support for destructuring](#support-for-destructuring)
* [auto generate specs for multiple arities](#specification-for-multiple-arities)
* [reference other specs](#reference-other-specs)
* [data DSL for homogeneous collections](#data-dsl-for-homogeneous-collections)
* [data DSL for tuples](#data-dsl-for-tuples)
* [global switch to toggle specification](#global-switch-to-toggle-specification)


### Specification at definition
```clojure
(ns me.lomin.sayang.api-test
  (:require [clojure.test :refer :all]
            [clojure.spec.alpha :as s]
            [me.lomin.sayang :as sg]
            [orchestra.spec.test :as orchestra])
  (:import (clojure.lang ExceptionInfo)))

(declare thrown?)
(s/check-asserts true)
(alter-var-root #'sg/*activate* (constantly true))

(sg/sdefn basic-usage {:ret    string?
                       :fn #(<= (:x (:args %))
                                (count (:ret %)))}
          [[x :- int?]]
          (str x))

(deftest basic-usage-test
  (is (= "1" (basic-usage 1)))

  (testing "Fails :fn spec"
    (is (thrown? ExceptionInfo
                 (basic-usage 2))))
  (testing "Fails :args spec"
    (is (thrown? ExceptionInfo
                 (basic-usage "5")))))

(orchestra/instrument `basic-usage)

(sg/sdefn int-identity {:args (s/cat :x int?)}
  [x]
  x)

(deftest specs-from-meta-map-test
  (is (= 100 (int-identity 100)))

  (testing "Fails :args spec"
    (is (thrown? ExceptionInfo
                 (int-identity "100")))))

(orchestra/instrument `int-identity)
```
### Partial specification
```clojure
(sg/sdefn partial-specs {:ret string?}
          [f
           [x :- int?]]
          (f x))

(deftest partial-specs-test
  (is (= "5" (partial-specs str 5)))

  (testing "Fails :args spec"
    (is (thrown? ExceptionInfo
                 (partial-specs str "5"))))
  (testing "Fails :ret spec"
    (is (thrown? ExceptionInfo
                 (partial-specs identity 5)))))

(orchestra/instrument `partial-specs)
```
### Support for destructuring
```clojure
(sg/sdefn sum-first-two-elements
  [[[a b] :- (s/tuple int? int? int?)]]
  (+ a b))

(deftest support-for-destructuring-test
  (is (= 5 (sum-first-two-elements [2 3 4])))

  (testing "Fails :args spec"
    (is (thrown? ExceptionInfo
                 (sum-first-two-elements [2 3])))))

(orchestra/instrument `sum-first-two-elements)
```
### Specification for multiple arities
```clojure
(sg/sdefn make-magic-string {:ret string?}
          ([[x :- int?]]
           (str x "?"))
          ([[x :- string?] [y :- string?]]
           (str x "?" y)))

(deftest multi-arity-test
  (is (= "2?" (make-magic-string 2)))
  (is (= "2?!" (make-magic-string "2" "!")))

  (testing "Fails :args spec for arity-2"
    (is (thrown? ExceptionInfo
                 (make-magic-string 2 "!")))))

(orchestra/instrument `make-magic-string)

(defn result-larger-than-min-arg-value? [spec]
  (< (apply min (vals (:0 (:args spec))))
     (:ret spec)))

(sg/sdefn add-map-values {:ret    int?
                          :fn result-larger-than-min-arg-value?}
  [[{:keys [a b c]} :- map?]]
  (+ a b c))

(deftest fn-spec-for-multi-arity-test
  (is (= -1 (add-map-values {:a 1 :b 0 :c -2})))

  (testing "Fails :fn spec"
    (is (thrown? ExceptionInfo
                 (add-map-values {:a -1 :b -2 :c -3})))))

(orchestra/instrument `add-map-values)
```
### Reference other specs
```clojure
(s/def ::number? number?)
(sg/sdefn number-identity [[x :- ::number?]]
          x)

(deftest reference-to-speced-keywords-test
  (is (= 2 (number-identity 2)))

  (testing "Fails :args spec"
    (is (thrown? ExceptionInfo
                 (number-identity "2")))))

(orchestra/instrument `number-identity)

(sg/sdefn call-with-7 [[f :- (sg/of make-magic-string)]]
          (f 7))

(deftest of-test
  (is (= "7?" (call-with-7 make-magic-string)))

  (testing "'identity' does not fulfill fdef of 'make-magic-string'"
    (is (thrown? ExceptionInfo
                 (call-with-7 identity)))))

(orchestra/instrument `call-with-7)
```
### Data DSL for homogeneous collections
```clojure
(sg/sdefn speced-add {:ret int?}
          [[xs :- [int?]]]
          (apply + xs))

(deftest every-spec-data-dsl-test
  (is (= 105 (speced-add (range 15))))

  (testing "1.0 is no integer, therefore xs is no homogeneous collection any more"
    (is (thrown? ExceptionInfo
                 (speced-add (cons 1.0 (range 15)))))))

(orchestra/instrument `speced-add)
```
### Data DSL for tuples
```clojure
(sg/sdefn sum-of-4-tuple {:ret float?}
          [[tuple :- [int? float? int? float?]]]
          (apply + tuple))

(deftest tuple-spec-data-dsl-test
  (is (= 10.0 (sum-of-4-tuple [1 2.0 3 4.0])))

  (testing "Fails tuple spec of :args"
    (is (thrown? ExceptionInfo
                 (sum-of-4-tuple [1 2 3 4])))))

(orchestra/instrument `sum-of-4-tuple)
```

## Limitations

* Does not work for ClojureScript, yet.

## About

Sayang (사양) means specification in Korean.

## License

Copyright © 2017 Steven Collins

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
