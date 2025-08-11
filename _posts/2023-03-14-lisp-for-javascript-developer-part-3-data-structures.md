---
layout: post
title: Lisp for JavaScript developer part 3 - data structures [JavaScript, Lisp]
date: 2023-03-14
tags: js lisp
series: jslisp
---

{%- include /_posts/series-toc.md series-name="jslisp" -%}

## Introduction 

This article will discuss typical functional data structures like immutable linked lists or maps, and a more Clojure-specific collection called a vector.

### Immutable singly-linked list

So far, we have seen only simple data types like `string`, `number`, or `boolean`. Now let's talk about collection data types. In functional programming, the most important and representable type of collection is an immutable singly-linked list. The list consists of nodes, each node holds two members: a single value of some type and a reference to the next node. As a simplification, we can now assume that the `nil` value presents an empty list. The list itself is just a reference to the first node in the list. It is an example of a recursive data structure, meaning it's defined in terms of itself.  In TypeScript, a sample type definition could look like this

```ts
type List<T> = { readonly head: T; readonly tail: List<T> | null;}
``` 

Very often, terms like `head` and `tail` are used once we talk about linked lists. Head represents the first element from the list, and tail represents the remaining list without the first element, potentially an empty list.

The crucial feature of functional data structures is immutability. We can not change a collection after its creation. Every time we insert, update, or delete items, something new is returned, leaving the original collection unchanged. Many names describe this behavior, such as "copy-on-write" or "non-destructive mutations."

The simplest way to explain this concept in JavaScript is to use a spread operator. Lets' say we have an array `var a = [5, 10, 15]` and we would like to add a new item `100` at the end. We can execute the built-in method `a.push(100)`, but this operation changes the original array, returning void (`undefined` to be precise).  Executing the following code `a = [...a, 100]` creates a new array in memory, copying all existing items and adding a new item at the end. And now there is a question: is there any difference between those two approaches? In both cases, after the operation, the variable `a` represents an array with a new item added. But in fact, there is a big difference. The variable `a` is just a reference to an array in memory. In the second approach, the array itself is not changed, only the reference to the new array in memory is set. Assume that before changing the collection, a new variable `b` is defined `var b = a`. Now two variables, `a` and `b`, point to the same collection. After executing `a = [...a, 100]`, variable `b` still points to the unchanged array holding values `[1,2,3]`. 

Another related concept is "persistent data structures." In the previous example, the execution of `[...a, 100]` allocates a new memory and copies all items. Adding another item to the collection takes the same steps as the beginning. So, every collection change generates a new memory that has to be cleaned up at some point. 

An immutable linked list represents a "persistent data structure", let's quote [wiki](https://en.wikipedia.org/wiki/Persistent_data_structure): 

> In computing, a persistent data structure or not ephemeral data structure is a data structure that always preserves the previous version of itself when it is modified

Let's take a look at the example, we have a list `var a = {head: 5, tail: {head: 10, tail: {head: 15, tail: null }}}` and we add a new item `a = {head: 5, tail: a}`. Adding an item to the beginning of the linked list creates a new `head` pointing to the `tail`, which is the list we are adding to. Removing the first item returns the `tail` list. Changing the list returns some new value, but no memory copying is involved. This feature is often called structure sharing. Of course, not all operations on the list can work this way. For instance, inserting a new item in the middle requires copying all items before the inserted item.  

Whether a collection is implemented as a persisted data structure or not is not obvious just by looking at the API of operations. In general, functional data structures don't mutate, so they always return some result; they are expressions instead of statements. In the Clojure dialect of Lisp, all built-in collections are implemented as persisted data structures.

There are two ways of creating a linked list in Clojure: a helper function `(list 1 2 3)` or a special syntax `'(1 2 3)`. We can also create a list by adding a new element at the beginning `(cons 1 (cons 2 '()))`. Of course, each list item can be of a different data type. Like all data types in functional programming, lists are compared by value, not by reference. Built-in function called `first` returns the `head` element, `(first '(1 2 3))` returns  `1`. Function `rest` returns the `tail`, `(rest '(1 2 3))` returns `'(2 3)`. 

```scheme
(def empty-list-1 '())
(def empty-list-2 (list))

(= empty-list-1 empty-list-2) ;; => true
(empty? empty-list-1) ;; => true

(def list-1 (cons 1 (cons 2 '())))
(def list-2 '(1 2))
(def list-3 (list 1 2))

(= list-1 list-2 list-3) ;; => true
(= '(1 (+ 1 1)) (list 1 (+ 1 1))) ;; => false
```

```js
var empty = { type: "list-empty" };
var cons = (head, tail) => ({ type: "list-cons", head, tail });

function lengthOfList(list) {
    return list.type === "list-empty" ? 0 : 1 + lengthOfList(list.tail);
}

var list = (...args) => args.reduceRight((tail, head) => cons(head, tail), empty);
var listp = coll => coll.type === "list-empty" || coll.type === "list-cons";
var conjToList = (coll, item) => cons(item, coll);

function* seqFromList(coll) {
    var node = coll;

    while (node.type !== "list-empty") {
        yield node.head;
        node = node.tail;
    }
}

var list1 = cons(1, cons(2, cons(3, (empty))));
var list2 = list(1, 2, 3);;
lengthOfList(list1); // => 3
listp(list()); // => true
```

The code above presents how easily an immutable linked list with basic operations can be implemented in JavaScript. This implementation is quite different from the TypeScript type definition provided at the beginning of this section. Each node in the list has an additional property `type` instead of using `null` for an empty list. For an empty list, this property is set to value `"list-empty"`. For non-empty lists, it is set to `"list-cons"`. This way we can differentiate between `null` value and an empty list. Helper function `seqFromList` creates a lazy sequence (JavaScript iterator object) from our linked list. 

#### Vector

An immutable linked list is a classic and most often used collection in functional languages; vector collection is more specific to the Clojure dialect of Lisp. Vector is immutable, compared by value, and implemented as a persistent data structure like all built-in collections in Clojure. In contrast to the list a new item is added to the end of the vector instead of the beginning. We can also efficiently access an element by its index. The vector is like an array collection known from other programming languages. New vector instance can be created using function `(vector 1 2)` or a special syntax using square brackets `[1 2]`. 

```scheme
(def vector-1 [1 2])
(def vector-2 (vector 1 2))
(= vector-1 vector-2) ;; => true
```

```js
var vector = (...args) => args;
var vectorp = coll => Array.isArray(coll);
var lengthOfVector = coll => coll.length;
var conjToVector = (coll, item) => [...coll, item];

vectorp(vector(1, 2, 3)); // => true
lengthOfVector(vector(1, 2, 3)); // => 3
```

The `Array` type was used to represent a vector collection because it provides fast access to an element by index.

#### Map

A map is another frequently used collection type, often called a dictionary or associative array. Each stored value is associated with some unique key. We can quickly find a specific value by its key. In Clojure, a map is defined using curly brackets like `{:name "marcin" :age 123}`, in this particular example, the keys are keywords `:name` and `:age`. The natural choice to represent map in JavaScript is an object. 

```scheme
(def map-1 {:name "marcin" :age 123})
(def map-2 {:age 123 :name "marcin"})
(= map-1 map-2) ;; => true
```

```js
var lengthOfMap = coll => Object.keys(coll).length;
function mapp(coll) {
    return !listp(coll) && !vectorp(coll) && !setp(coll) && !functionp(coll)
        && !numberp(coll) && !stringp(coll) && !functionp(coll) && !nilp(coll);
}
function conjToMap(coll, item) {
    var [key, value] = Array.isArray(item) ? item : Object.entries(item)[0];
    return { ...coll, [key]: value };
}

var map1 = { name: "marcin", age: 123 };
mapp({ name: "marcin" }); // => true
lengthOfMap({ name: "marcin" }); // => 2
```


#### Set

A set collection stores unique values. We can use a helper function `hash-set` or a special syntax ` #{1 2}` to create an instance of a set. In JavaScript, a set collection can be represented as the built-in type `Set`.

```scheme
(def set-1 #{1 2})
(def set-2 (hash-set 1 2))
(= set-1 set-2) ;; => true
```

```js
var set = items => new Set(items);
var hashSet = (...items) => set(items);
var setp = coll => coll instanceof Set;
var lengthOfSet = coll => coll.size;
var conjToSet = (coll, item) => coll.has(item) ? coll : new Set([...coll, item]);

var set1 = set([1, 2, 3, 1]);
var set2 = hashSet(1, 2, 3, 1);
setp(hashSet(1, 2, 3, 4)); // => true
lengthOfSet(hashSet(1, 2, 3)); // => 3
```

#### Seq

All collection types in Clojure can be converted to the Seq type; a linked list is treated as a Seq by default without explicit conversion. It is not the perfect analogy, but we can think about the Seq collection as an API describing the Iterator pattern. It allows lazy iteration over a sequence of items. Clojure implements many common functions in terms of the Seq type, so internally, the specific collection is converted into the Seq type, and then common logic based on the Seq is executed.  

```scheme
(seq? "abc") ;; => false
(seq? (seq "abc")) ;; => true
(seq? (vector 1 2 2)) ;; => false
(seq? (seq (vector 1 2 2))) ;; => false
(seq? (list 1 2 3)) ;; => true
```

```js
var seqp = coll => typeof coll[Symbol.iterator] !== "undefined";
function seq(coll) {
    return seqp(coll) ? coll // string, array, set
        : listp(coll) ? seqFromList(coll)
            : mapp(coll) ? Object.entries(coll)
                : wrongArgType(coll);
}
```

We can use a JavaScript language feature called iterators to represent Seq. Most built-in types, like `Array, Set, String,` implement iterators directly. For other custom types, like linked lists, we can provide a simple conversion function using JavaScript generators (functions using the `yield` keyword). 

#### Collection operations

Up to this moment, we were primarily focused on the general characteristics of different collection types, but not necessarily on the operations. In functional programming, we separate the data from the behavior. In object-oriented programming, the definition of a class combines data (fields and properties) with behavior (methods). In Clojure, the same functions work with many different types of collections. For example, `count` returns the number of all elements, `empty?` checks whether the collection is empty, `first` returns the first element. `conj` stands for "conjoining" and inserts a new item into the collection. Still, the behavior is different depending on the collection type. For a linked list, a new item is added at the beginning; for a vector, an item is added at the end. The `into` function takes two collections and inserts all items from the second collection into the first using the `conj` function.

```scheme
(count list-1) ;; => 2
(count vector-1) ;; => 2
(count map-1) ;; => 2
(count set-1) ;; => 2

(conj list-1 3 4 5) ;; => (5 4 3 1 2)
(conj vector-1 3 4 5) ;; => [1 2 3 3 4 5]
(conj set-1 3 4 5 1) ;; => #{1 4 3 2 5}

(into list-1 [3 4 5]) ;; => (5 4 3 1 2)
(into vector-1 [3 4 5]) ;; => [1 2 3 3 4 5]
(into set-1 [3 4 5 1]) ;; => #{1 4 3 2 5}

(let [coll vector-1]
  [(first coll)
   (rest coll)
   (empty? coll)
   (nth coll 1)]) ;; => [1 (2) false 2]
```

```js
function count(coll) {
    return listp(coll) ? lengthOfList(coll)
        : vectorp(coll) ? lengthOfVector(coll)
            : setp(coll) ? lengthOfSet(coll)
                : mapp(coll) ? lengthOfMap(coll)
                    : wrongArgType(coll);
}

function conj(coll, ...items) {
    const conjItem = listp(coll) ? conjToList
        : vectorp(coll) ? conjToVector
            : setp(coll) ? conjToSet
                : mapp(coll) ? conjToMap
                    : wrongArgType(coll)

    return items.reduce(conjItem, coll);
}

var into = (coll, items) => conj(coll, ...items);

into(list(1, 2, 3), vector(4, 5));

var first = coll => find_(seq(coll));
var rest = coll => listp(coll) ? coll.tail : skip_(seq(coll), 1);
var nth = (coll, n) => elementat_(seq(coll), n);
var emptyp = coll => !some_(seq(coll));
```

All functions named with the suffix `_`, like `find_, skip_, some_, ...` were imported from [poweseq](https://github.com/marcinnajder/powerseq) library at the beginning of this series of articles.

#### Map operations

The most commonly used map operation is getting the value for a specified key. It can be achieved by calling a helper function`(get map-1 :name)`, where `map-1` is a map and `:name` is a key. Both arguments can be treated as a function, so the following calls `(map-1 :name), (:name map-1)` may look strange but are entirely correct. It's equivalent to calling the`get` function. The `assoc` function adds, and the `dissoc` function removes one or many items from the map. The `update` function takes a key and a function that calculates a new value from the old value. Some functions work not only with map type but also with vector or string, in such cases the index of the item is treated as a key in the map.


```scheme
(get map-1 :name) ;; => "marcin"
(map-1 :name) ;; => "marcin" (... map is a function)
(:name map-1) ;; => "marcin" (... keyword is a function)

(assoc map-1 :address "wroclaw" :id 1) ;; => {:name "marcin", :age 123, :address "wroclaw", :id 1}
(dissoc map-1 :name :id) ;; => {:age 123}

(assoc map-1 :name "lukasz") ;; => {:name "lukasz", :age 123}

(update map-1 :name (fn [value] (str value "!"))) ;; => {:name "marcin!", :age 123}
(update map-1 :name str "!") ;; => {:name "marcin!", :age 123}
```

```js
var get = (coll, key) => vectorp(coll) || stringp(coll) || mapp(coll) ? coll[key] : wrongArgType(coll);

function update(coll, key, func, ...args) {
    return vectorp(coll) ? coll.map((e, i) => i === key ? func(e, ...args) : e)
        : mapp(coll) ? Object.entries({ [key]: null, ...coll }).reduce((o, [k, v]) => ({ ...o, [k]: k === key ? func(v, ...args) : v }), {})
            : wrongArgType(coll);
}

function assoc(coll, ...kvs) {
    if (nilp(coll) || kvs.length === 0) {
        return coll;
    }

    const [key, value, ...other] = kvs;

    const coll2 = (vectorp(coll) || stringp(coll)) && (key < 0 || key >= coll.length) ? throww("index out of bounds")
        : stringp(coll) ? coll.substring(0, key) + value + coll.substring(key + 1)
            : update(coll, key, _ => value);

    return assoc(coll2, ...other);
}


function dissoc(coll, ...keys) {
    const keysSet = new Set(keys);
    return mapp(coll)
        ? Object.entries(coll).reduce((o, [k, v]) => keysSet.has(k) ? o : { ...o, [k]: v }, {})
        : wrongArgType(coll);
}


var p1 = { id: 1, name: "marcin", age: 123, address: { city: "wroclaw", country: "poland" } };

get(p1, "name"); // => "marcin"
get(vector(1, 2, 3), 1); // 2
assoc(p1, "age", 5, "address", "wroclaw"); // => { address: 'wroclaw', age: 5, id: 1, name: 'marcin' }
update(vector(1, 2, 3), 0, v => v + 10); // => [ 11, 2, 3 ]
update(vector(1, 2, 3), 0, plus, 10, 5); // => [ 11, 2, 3 ]
dissoc(p1, "name", "age"); // => { id: 1 }
```

##### Nested maps

In most programming languages there is a distinction between describing an object of particular shape (e.g., class or record type "Person" with fields "name" and "age") and a typical mapping data structure for fast search by key (e.g. list of countries searched by unique code `Map<string, CountryInfo>`). In Clojure, the same map data structure is used for both scenarios. Let's say we want to store informacje about line in 2D space, each line has two points with x and y coordinates `{:start {:x 1 :y 2} :end {:x 10 :y 20}}`. It requires nesting one map inside the other map. We must remember that all operations over data structures in Clojure are immutable. If we want to change `:x` coordinate, a new updated point will be created, and a new outer object representing the line must also be created. Clojure provides a special group of functions like `get-in, assoc-in, update-in` to simplify the management of nested data structures. 

```scheme
(def line {:start {:x 1 :y 2} :end {:x 10 :y 20}})

(get-in line [:start :x]) ;; => 1
(assoc-in line [:start :x] 0) ;; => {:start {:x 0, :y 2}, :end {:x 10, :y 20}}

(update-in line [:start :x] inc) ;; => {:start {:x 2, :y 2}, :end {:x 10, :y 20}}
(update-in line [:start :x] + 1) ;; => {:start {:x 2, :y 2}, :end {:x 10, :y 20}}
```

```js
function getIn(coll, ks) {
    const [key, ...other] = ks;
    return nilp(coll) || ks.length === 0 ? coll : getIn(get(coll, key), other);
}


function updateIn(coll, ks, func, ...args) {
    const [key, ...other] = ks;
    return nilp(coll) || ks.length === 0 ? coll
        : ks.length === 1 ? update(coll, key, func, ...args)
            : update(coll, key, v => updateIn(nilp(v) ? {} : v, other, func, ...args));
}

function assocIn(coll, ks, value) {
    return updateIn(coll, ks, _ => value, value);
}

getIn(p1, ["address", "city"]);
getIn(p1, ["address", "city", 0]); // => w

updateIn(p1, ["name"], x => x + "!");
updateIn(p1, ["lastName"], x => x === null ? "null" : "!null");
updateIn(p1, ["address", "city"], city => city.toUpperCase());
updateIn(p1, ["address", "postalCode"], _ => 666);
updateIn([p1, p1], [0, "name"], name => name + "!");

assocIn(p1, ["name"], "wojtek");
assocIn(p1, ["lastName"], "w");
assocIn(p1, ["address", "city"], "Krakow");
assocIn(p1, ["address", "postalCode"], 666);
assocIn([p1, p1], [0, "name"], "wojtek");
assocIn(assocIn({}, ["C", "photos", "image1.jpg"], 11111), ["C", "photos", "image2.jpg"], 22222);
// => { C: { photos: { 'image2.jpg': 22222, 'image1.jpg': 11111 } } }
updateIn(p1, ["address"], dissoc, "city")
```


All functions take a full path as a list of keys and perform some operation, even if some key in the middle of the path does not exist. It's unbelievably convenient in practice.

##### Map operations and vectors

As we mentioned before, the functions work with different types of collections. It's very common for different types of collections to be nested inside each other, like a vector containing many maps.

```scheme
(def users [{:name "James" :age 26}  {:name "John" :age 43}])
(get users 1) ;; => {:name "John", :age 43}
(get-in users [1 :name]) ;; => "John"
(update-in users [1 :age] inc) ;; => [{:name "James", :age 26} {:name "John", :age 44}]
```

#### Seq operators (map, filter, reduce, ... )

Nowadays, almost any programming language supports functions like `map, filter, reduce`, and many other functions working with collections. It's worth to remember that the Lisp language invented them. One of the key features of functional programming is composition. We can build bigger expressions from smaller ones, like calling the `filter` function and then passing the result into the `map` function. Those functions take and return a Seq type, so they work lazily. Clojure supports a shorthand way of writing a lambda expression. Instead of writing `(fn [x] (* x 10))`, we can write `#(* % 10`). 

```scheme
(map
 (fn [x] (* x 10))
 (filter odd? [1 2 3 4 5 6])) ; => (10 30 50)

(map
 #(* % 10)
 (filter odd? [1 2 3 4 5 6])) ; => (10 30 50)
```

```js
var map = (f, coll) => map_(seq(coll), f);
var filter = (f, coll) => filter_(seq(coll), f);
var repeat = (n, x) => repeatvalue_(x, n);

var reducedS = Symbol();
var reduced = v => ({ s: reducedS, v });
function reduce(f, val, coll) {
    return iter(seq(coll)[Symbol.iterator](), val);

    function iter(iterator, total) {
        var { done, value } = iterator.next();
        if (done) {
            return typeof value === "undefined" ? total : f(total, value);
        }
        var t = f(total, value);
        return t.s === reducedS ? t.v : iter(iterator, t);
    }
}

map(filter(oddp, list(1, 2, 3, 4, 5, 6)), x => x * 10);

reduce((p, c) => p + c, 0, list());
reduce((p, c) => p + c, 0, list(1, 2, 3));
reduce((p, c) => (c === 2 ? reduced(1000 + p) : p + c), 0, list(1, 2, 3));
reduce((p, c) => p + c, 0, (function* () {
    yield 10;
    yield 20;
    return 30;
})());
```

Clojure also has a not-so-well-known feature of the `reduce` function. In most languages, we can not stop the execution of `reduce`. In Clojure, it can be stopped by wrapping the final result value into a function call `(reduced final-result)`. Let's look at the example from the official documentation of [reduce](https://clojuredocs.org/clojure.core/reduce#example-55e60cdbe4b072d7f27980f5) 

```scheme
(defn limit [x y] 
  (let [sum (+ x y)] 
    (if (> sum 10) (reduced sum) sum)))

(reduce limit 0 (range 10)) ;; => 0+1+2+3+4+5 = 15
```

#### Summary 

Clojure, like any other dialect of Lisp, is a very minimalistic programming language. We don't need classes or OOP ways of solving problems. Data is stored in collections like lists or maps, separately a set of simple yet powerful functions. That's enough to solve real-world problems. 