---
title: Clojure
published: April 6, 2014
excerpt: A lisp-inspired JVM language
comments: off
toc: left
---

Recently I've been thinking about my opinions on the various languages I know, particularly with regards to which I should focus on, and I decided that knowing a JVM language would be very beneficial because of how robust and time-proven the JVM is, especially compared to other language VMs. For this reason I considered Scala and Clojure, and Scala seemed more like Haskell to me so I decided to [go with that one first].

[go with that one first]: /notes/scala

*[JVM]: Java Virtual Machine
*[VM]: Virtual Machine

I didn't have an overwhelming reaction to Scala, so I decided to give Clojure a shot as well to better compare them. My main resource is the book [Clojure Programming]. There's also [Clojure for the Brave and True].

[Clojure Programming]: http://amzn.com/1449394701
[Clojure for the Brave and True]: http://www.braveclojure.com/

* toc

# Homoiconicity

Clojure and other lisp languages are _homoiconic_, often referred to as "code as data," which means that the way the code is written is itself the abstract syntax tree (AST) of the program. This makes creating embedded domain specific languages (EDSLs) very straightforward, as well as simply making the code easier to reason about. Questions of precedence, for example, are directly encoded into the code.

*[AST]: Abstract Syntax Tree
*[EDSL]: Embedded Domain Specific Language

Because Clojure code is itself an AST in a Clojure data structure, metaprogramming is also more powerful because it simply involves manipulating that data structure; this is the basis of macros.

## Reader

Clojure AST structures can be deserialized into Clojure structures using the `read`-like functions. In the following examples, the structures are printed back out by the REPL using the `pr-str` function. The fact that serialization of Clojure structures is this straightforward is what drives most Clojure developers to use it as the primary serialization mechanism.

*[REPL]: Read-Eval-Print-Loop

``` clojure
(read-string "42")
;= 42
(read-string "(+ 1 2)")
;= (+ 1 2)
(pr-str (read-string "[1 2 3]"))
;= [1 2 3]
```

<div class="callout">

**Note**: the Clojure REPL always starts in the default `user` namespace.

</div>

The reader allows for different syntax to make code more concise. For example, evaluation of a form can be suppressed by prefixing it with a quote `'`. Anonymous function literals can be defined with `#()`.

## Scalar Literals

Characters are denoted by a blackslash, as in `\c`, and they natively support Unicode. It's also possible to use special characters such as `\n` would be used in strings, but individually:

* `\space`
* `\newline`
* `\formfeed`
* `\return`
* `\backspace`
* `\tab`

### Keywords

Keywords I believe are similar to Ruby/Scala symbols and Erlang atoms. The are prefixed by a colon `:` and consist of any non-whitespace character, where a slash `/` denotes a _namespaced keyword_, and a double colon `::` is expanded by the reader to a namespaced keyword in the current namespace, or another namespace if the keyword started by a namespace alias as in `::alias/keyword`.

``` clojure
(def pizza {:name "Ramunto's"
            :location "Claremont, NH"
            ::location "123,-456"})
;= #'user/pizza
pizza
;= {:name "Ramunto's", :location "Claremont, NH", :user/location "123,-456"}
(:user/location pizza)
;= "123,-456
```

Keywords are "named" values which are values that have intrinsic names that can be accessed using the `name` function, and the namespace can be accessed with the `namespace` function:

``` clojure
(name :user/location)
;= "location"
(namespace :user/location)
;= "user"
```

As in Ruby, keywords are often used for indexing hashes. The following defines a hashmap with a `:name` and `:city` key and then accesses the value for the `:city` key. Keywords can be used in the function position because they _are_ functions that look themselves up in collections passed to them.

``` clojure
(def person {:name "Sandra Cruz"
             :city "Portland, ME"})
;= #'user/person
(:city person)
;= "Portland, ME"
```

### Symbols

Symbols are identifiers that evaluate to the values they name. For example, in the following code, `average` is a symbol referring to the function held in the var named `average`. Symbols containing a slash `/` denote a _namespaced symbol_ which evaluates to the named value in the specified namespace.

``` clojure
(average [10 20 30])
;= 20
```

The variables can be referred to directly by prefixing a symbol with `#'`.

### Numbers

Numeric literals exist for a variety of number types. Custom numerical bases can be used with the `#r` prefix where `#` would be the desired number base.

Syntax               Type
-------              -----
42, 0xff, 2r101, 040 long
3.14, 6.02e23        double
42N                  clojure.lang.BigInt
0.01M                java.math.BigDecimal
22/7                 clojure.lang.Ratio

### Regular Expressions

Strings prefixed with a hash `#` are regex literals which yield `java.util.regex.Pattern` instances.

``` clojure
(re-seq #"(\d+)-(\d+)" "1-3")
;= (["1-3" "1" "3"])
```

### Comments

Single-line comments are started with a semicolon `;`. There are also _form-level_ comments prefixed by the `#_` reader macro which cue the reader to ignore the next Clojure _form_ following the macro. This is particularly useful when wanting to comment out blocks of code. The `comment` macro can also be used to comment out code but they always evaluate to `nil`, which may be unexpected.

``` clojure
(read-string "(+ 1 2 #_(* 2 2) 8)")
;= (+ 1 2 8)

(defn some-func
  [args]
  code
  #_(if debug-level
    (println "debugging")
    (println "more debugging")))

(+ 1 2 (comment (* 2 2)) 8)
;= NullPointerException
```

### Whitespace

Commas are considered whitespace by the reader. Whether to use them or not is a question of style, but they're generally used when multiple pairs of values appear on the same line.

``` clojure
(= [1 2 3] [1, 2, 3])
;= true

(create-user {:name user, :email email})
```

### Collections

There are literals for lists, vectors, maps, and sets. Note that since lists denote calls in Clojure, it's necessary to quote them to prevent their evaluation as a call.

``` clojure
'(a b :name 12.5)     ;; list
['a 'b 12.5]          ;; vector
{:name "Bob" :age 31} ;; map
#{1 2 3}              ;; set
```

## Namespaces

Vars are defined using the `def` special form which takes the symbol used to refer to the var and the value to store in that var. When the symbol is used on its own to access the var's value, the symbol is said to be _unqualified_ and so it is resolved within the current namespace. Vars can also be redefined by supplying the same symbol with a different value to the `def` function.

``` clojure
(def x 1)
;= #'user/x
x
;= 1
(def x "hello")
;= #'user/x
x
;= "hello"
```

Contrary to unqualified symbols, symbols can be _namespace-qualified_ so that they are resolved within the specified namespace. For example, if we create a new namespace `foo`, we can continue to refer to the symbol `x` by qualifying the namespace:

``` clojure
*ns*
;= #<Namespace user>

(ns foo) ; create and switch to new 'foo' namespace
;= nil

*ns*
;= #<Namespace foo>

user/x
;= "hello"
```

However, if we attempt to access the unqualified symbol, it will try to find `x` within the `foo` namespace, which doesn't exist:

``` clojure
x
;= CompilerException: Unable to resolve symbol: x
```

## Special Forms

Special forms are Clojure's primitive building blocks of computation upon which the rest of Clojure is built.

### Suppressing Evaluation

The special form `quote` suppresses evaluation of a Clojure expression. For example symbols evaluate to the value of the var they represent, but with `quote` that evaluation is suppressed, so they evaluate to themselves like strings and numbers do. The quote character `'` is reader syntax for `quote`. In fact, `quote` can be used on reader sugars to determine how they're actually represented.

``` clojure
(quote x)
;= x
(symbol? (quote x))
;= true

'x
;= x

(symbol? 'x)
;= true

(= '(+ x x) (list '+ 'x 'x))
;= true
```

### Code Blocks

The special form `do` evaluates all of the expressions provided to it in order and yields the last expression's value as its value. Many other forms such as `fn`, `let`, `loop`, `try` and `defn` wrap their body in an implicit `do` expression so that multiple inner expressions are evaluated.

``` clojure
(do
  (println "hi")
  (apply * [4 5 6]))
; hi
;= 120

(let [a (inc (rand-int 6))
      b (inc (rand-int 6))]
  (println (format "You rolled a %s and a %s" a b))
  (+ a b))
```

### Vars

The special form `def` defines or redefines a var with an optional value within the current namespace. Other forms implicitly create or redefine vars and are usually prefixed with `def` such as `defn` and `defn-`.

It's possible to refer to vars instead of the values that they hold by using the special form `var`. There's also a shorthand for this with `#'`.

``` clojure
(def x 5)
;= #'user/x

(var x)
;= #'user/x

#'x
;= #'user/x
```

### Local Bindings

The special form `let` allows lexically scoped named references to be defined.

``` clojure
(defn hypot
  [x y]
  (let [x2 (* x x)
        y2 (* y y)]
    (Math/sqrt (+ x2 y2))))
```

The `let` form also allows _destructuring_ similar to pattern-matching in languages like Haskell, Rust, Scala, and Erlang. For example, to destructure a sequence, specifically a vector, we simply pass it a list of symbols that will take on the appropriate values. Destructuring can also be nested, as in other languages. The ampersand `&` can be used to specify that the following symbol should take on the remaining _sequence_ of values. The `:as` keyword can be used to bind the collection to a value, similar to what `@` does in Haskell, Scala, and Rust.

``` clojure
(def v [42 "foo" 99.2 [5 12]])
;= #'user/v

(let [[x y z] v]
  (+ x z))
;= 141.2

(let [[x _ _ [y z]] v]
  (+ x y z))
;= 59

(let [[x & rest] v]
  rest)
;= ("foo" 99.2 [5 12])

(let [[x _ z :as original-vector] v]
  (conj original-vector (+ x z)))
;= [42 "foo" 99.2 [5 12] 141.2]
```

Maps can also be destructured in a similar manner. This works with Clojure's `hash-map`, `array-map`, records, collections implementing `java.util.Map`, and values supported by the `get` function such as Clojure vectors, strings, and array can be keyed by their indices.

``` clojure
(def m {:a 5 :b 6
        :c [7 8 9]
        :d {:e 10 :f 11}
        "foo" 88
        42 false})
;= #'user/m

(let [{a :a b :b} m]
  (+ a b))
;= 11

(let [{x 3 y 8} [12 0 0 -18 44 6 0 0 1]]
  (+ x y))
;= -17
```

The `:as` keyword can be used to bind the collection. The `:or` keyword can be used to provide a defaults map which will be consulted if the destructured keys aren't present.

``` clojure
(let [{k :unknown x :a
       :or {k 50}} m]
  (+ k x))
;= 55
```

Often times it may be desirable to destructure a map such that the symbols are named after the keys of the map, but doing this explicitly can get repetitive, which is why the options `:keys`, `:strs`, and `:syms` can be used.

``` clojure
(def chas {:name "Chas" :age 31 :location "Massachusetts"})
;= #'user/chas

(let [{name :name age :age location :location} chas]
  (format "%s is %s years old and lives in %s." name age location))

(let [{:keys [name age location]} chas]
  (format "%s is %s years old and lives in %s." name age location))
```

It's also possible to destructure vectors which themselves contain key-value pairs. This can be done explicitly by binding the key-value pairs with `&`, converting that to a `hash-map`, and then destructuring that --- but it's also possible with regular destructure syntax. This is specifically made possible by `let` by allowing the destructuring of rest sequences if they have an even number of values, i.e. key-value pairs.

``` clojure
(def user-info ["robert8990" 2011 :name "Bob" :city "Boston"])
;= #'user/user-info

(let [[username account-year & extra-info] user-info
      {:keys [name city]} (apply hash-map extra-info)]
  (format "%s is in %s" name city))
;= "Bob is in Boston"

(let [[username account-year & {:keys [name city]}] user-info]
  (format "%s is in %s" name city))
;= "Bob is in Boston"
```

### Creating Functions

The special form `fn` is used to create functions. A function defined this way has no name, and so cannot be referred to later on. It can be place inside a var using the `def` form. The `fn` form also takes an optional name by which the function can reference itself. Furthermore, a function can have _multiple arities_, that is, define different bodies depending on the number of arguments passed.

``` clojure
((fn [x] (+ 10 x)) 8)

(def add-ten (fn [x] (+ 10 x)))

(add-ten 20)
;= 30

(def strange-adder (fn adder-self-reference
                    ([x] (adder-self-reference x 1))
                    ([x y] (+ x y))))

(strange-adder 10)
;= 11

(strange-adder 10 50)
;= 60
```

The `defn` form encapsulates the functionality of `def` and `fn`.

``` clojure
(defn strange-adder
  ([x] (strange-adder x 1))
  ([x y] (+ x y)))
```

The special form `letfn` can be used to define multiple functions at once that are aware of each other. This is useful for definining mutually recursive functions.

``` clojure
(letfn [(odd? [n]
          (even? (dec n)))
        (even? [n]
          (or (zero? n)
            (odd? (dec n))))])

(odd? 11)
;= true
```

Variadic functions are possible using the rest arguments syntax from destructuring.

``` clojure
(defn concat-rest
  [x & rest]
  (apply str (butlast rest)))

(defn make-user
  [& [user-id]]
  {:user-id (or user-id (str (java.util.UUID/randomUUID)))})
```

It's also possible to use keyword arguments in functions, which is facilitated through map destructuring of rest sequences.

``` clojure
(defn make-user
  [username & {:keys [email join-date]
               :or {join-date (java.util.Date.)}}]
  {:username username
   :join-date join-date
   :email email
   :exp-date (java.util.Date. (long (+2.592e9 (.getTime join-date))))})

(make-user "Bobby")
(make-user "Bobby"
            :join-date (java.util.Date. 111 0 1)
            :email "bobby@example.com")
```

Function literals have specific, concise syntax by being prepended with `#`. Placeholder arguments are prepended with `%`, though the first argument can be referred to by a single `%`. Function literals don't contain an implicit `do` form, so multiple statements require an explicit `do` form. It's also possible to specify variadic functions by assigning the rest of the arguments to `%&`.

``` clojure
(fn [x y] (Math/pow x y))
#(Math/pow %1 %2)

(fn [x & rest] (- x (apply + rest)))
#(- % (apply + %&))
```

<div class="callout">

**Note**: Function literals cannot be nested.

</div>

### Conditionals

The special form `if` is the single primitive conditional operator in Clojure. If no else-expression is provided it is assumed to be `nil`. There are other conditionals based on this form that are more convenient in specific situations.

``` clojure
(if condition? true false)

(if-let [nums (seq (filter even? [1 2 3 4]))]
  (reduce + nums)
  "No even numbers found.") ; else

(when (= x y) true) ; else nil

(cond
  (< n 0) "negative"
  (> n 0) "positive"
  :else   "zero")
```

### Looping

The special form `recur` transfers control to the local-most `loop` or function, allowing recursion without consuming stack space and thereby overflowing the stack. The `loop` special form takes a vector of binding names and initial values. The final expression is taken as the value of the form itself. The `recur` special form is considered very low-level that is usually unnecessary, instead opting for `doseq`, `dotimes`, `map`, `reduce`, `for`, and so on.

``` clojure
(loop [x 5]
  (if (neg? x)
    x
    (recur (dec x))))

(defn countdown
  [x]
  (if (zero? x)
    :blastoff!
    (do (println x)
        (recur (dec x)))))
```

### Java Interop

The special forms `.` and `new` exist for Java interoperability. Their use is somewhat unnatural in Clojure, however, and so there are sugared forms which are idiomatic.

There are also special forms for exception handling and throwing. There are also lock primitives to synchronize on the monitor associated with every Java object, but this is usually unnecessary as there's macro `locking` that is better suited.

``` clojure
; instantiation
(new java.util.ArrayList 100)
(java.util.ArrayList. 100)

; static method
(. Math pow 2 10)
(Math/pow 2 10)

; instance method
(. "hello" substring 1 3)
(.substring "hello" 1 3)

; static fields
(. Integer MAX_VALUE)
Integer/MAX_VALUE

; instance field
(. some-object some-field)
(.someField some-object)
```

### Specialized Mutation

The `set!` special form can be used to perform in-place mutation of state, which is useful for setting thread-local values, Java fields, or mutable fields.

### Eval

The `eval` form evaluates its single argument form, which is useful when used with `quote` or `'` to suppress evaluation of the argument until it's evaluated by `eval`. With this final form, it's possible to reimplement a simple REPL.

``` clojure
(defn simple-repl
  "Simple REPL. :quit to exit."
  []
  (print (str (ns-name *ns*) ">>> "))
  (flush)
  (let [expr (read)
        value (eval expr)]
    (when (not= :quit value)
      (println value)
      (recur))))

(simple-repl)
```

