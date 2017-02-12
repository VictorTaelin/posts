## Foldr is like search-replace, but for list constructors.

One of the major sources of confusion for newcomers to functional programming languages is the lack of loops everywhere. They are there, but rarely mentioned. This often leaves the newcomer with a sense of impotence, because "how are you supposed to do anything at all without loops"? The common answer is: `recursion`; but that answer is suspect, because *recursion is hard*. It has most of the same problems that loops have and is equally tricky to analyze. Recursion can, yes, replace loops in the general case, but, in practice, what really replaces `loops` for most cases are `folds`, which are much simpler. Here, I'll try to explain `folds` in a friendly way, consisting mostly of examples and simple analogies, and using a widely known language, JavaScript, in order to avoid unnecessary friction. In order to do so, I'll talk about `foldr`, which is `fold` applied to lists specifically. 

> Foldr is just like search-replace, but for list constructors.

The intuition above isn't a dumbed-down analogy (like "Monads are burritos"), it is an actually accurate description of how `foldr` works. Note that, in order for this to make sense, we must be talking about the same kind of "constructor":

> A "constructor", in a functional-programming context, is just a fancy name for "a tuple of values with a tag specifying its format".

For example, this is how the list `[1,2,3]` is expressed in a functional language:

```js
["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]]
```

`Cons` means *"this contains a value and another list"*, and `Nil`, means *"this is the end of the list"*. If you understand JSON, you understand constructors and, thus, functional datatypes. Once you accept that, then understanding `foldr` is trivial:

> `foldr` is just a function that takes a list and replaces every `["Cons", X, ...]` by `C(X, ...)`, and every `["Nil"]` by `N`, where `C` and `N` are anything you want.

And that is sufficient to implement any list-processing algorithm! Other datatypes have similar folds.

### Example 0: summing

We have a list like this:

```js
["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]]
```

And we want to sum every element. We can't use loops or recursion. `foldr` gives us the ability to replace every `["Cons", X, ...]` and every `["Nil"]` by anything we want. Notice we can solve this problem by replacing the former by `add(X, ...)` and the later by `0`:

```js
foldr(

  // replace ["Cons", X, ...] by add(X, ...)
  (x, _) => add(x, _),

  // and replace ["Nil"] by 0
  0,

  // on the list
  ["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]])
```

This is what happens:

```js

["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]]
-> add(1, add(2, add(3, 0)))
-> 6
```

This looks simple, and it is. Every `foldr`-based algorithm follows the same strategy: *"figure out what to put in place of `Cons X _` and `Nil` such that in the end you'll have what you want"*!

### Example 1: mapping

You can also use `foldr` instead of a loop to sum all elements of an array. Just replace `Cons X ...` by `Cons F(X) ...`!

```js
foldr(

  // replace ["Cons", X, ...] by ["Cons", F(X), ...]
  (x, _) => ["Cons", F(x), _], 

  // and replace ["Nil"] by ["Nil"]
  ["Nil"],                       

  // on the list
  ["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]])
```

This is what happens:

```js
["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]]
-> ["Cons", F(1), ["Cons", F(2), ["Cons", F(3), ["Nil"]]]]
```

### Example 2: filtering

You can use `foldr` instead of a loop to filter all elements of an array. Just replace `Cons X ...` by `COND(x) ? Cons X ... : ...`!

```js
foldr(

  // replace ["Cons", X, ...] by COND(X) ? ["Cons", X, ...] : ... 
  (x, xs) => x === 2 ? ["Cons", x, xs] : xs,

  // replace ["Nil"] by ["Nil"]
  ["Nil"],

  // on the list
  ["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]])
```

This is what happens:

```js
  ["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]]
  -> 1 === 2 ? ["Cons", 1, 2 === 2 ? ["Cons", 2, 3 === 2 ? ["Cons", 3, ["Nil"]] : ["Nil"]]
                                                 3 === 2 ? ["Cons", 3, ["Nil"]] : ["Nil"]]
             :             2 === 2 ? ["Cons", 2, 3 === 2 ? ["Cons", 3, ["Nil"]] : ["Nil"]]
                                   :             3 === 2 ? ["Cons", 3, ["Nil"]] : ["Nil"]]
  -> ["Cons", 2, ["Nil"]]
```

This is how you implement filter.

### Example 3: reversing

This is the trickiest because involves a new technique, passing state down. It works this way: instead of returning a reversed list directly, we return a function that receives a list, appends an element to it, and passes it down the chain all the way down to `["Nil"]`, which just returns the result. Take a look:

```js
foldr(
    
    // replace ["Cons", X, ...] by `list => _(["Cons", X, ...])`
    // (i.e., receive a new list, append X, send down)
    (X, _) => list => _(["Cons", X, list]),

    // replace ["Nil"] by `list => list`
    // (i.e., and of the chain, return the result)
    list => list,

    // on the list
    ["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]])

    // and call it with a new list
    (["Nil"])
```

This is what happens:

      
```js
["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]]
-> (list =>
     (list =>
       (list =>
         (list => 
           list)
         (["Cons", 3, list))
       (["Cons", 2, list]))
     (["Cons", 1, list]))
   (["Nil"])
-> ["Cons", 3, ["Cons", 2, ["Cons", 1, ["Nil"]]]]
```

This is how you implement list-processing functions which involve passing a state through the list.

### Conclusion

Hopefully this intuition makes `foldr` simpler than it is often portrayed. I do believe that, once you get it, folds are much easier to use than loops because you don't have to deal with indices and the like.

For completion and in case you want to explore, here is an working, JavaScript implementation of the concepts presented above:

```js
const foldr = cons => nil => list =>{
  switch (list[0]){
    case "Cons": return cons(list[1], foldr (cons) (nil) (list[2]));
    case "Nil": return nil;
  }
};

// Applies `f` to every element in a list
const map = f => foldr
  // replace all ["Cons",x,_] by ["Cons",f(x),_]
  ((x,_) => ["Cons", f(x), _])
  // replace Nil by Nil
  (["Nil"]);

// Removes every element `x` for which `cond(x)` is false
const filter = cond => foldr
  // replace all ["Cons",x,_] by (cond ? ["Cons",x,_] : _)
  ((x,_) => cond ? ["Cons", x, _] : _)
  // replace Nil by Nil
  (["Nil"]);

// Reverses the list
const reverse = list => foldr
  // replace all ["Cons",x,_] by (list => _(["Cons",x,list]))
  ((x,_) => list => _(["Cons", x, list]))
  // replace ["Nil"] by (list => list)
  (list => list)
  // on this list
  (list)
  // at this point we have a "reverser" machine: it receives
  // an empty list, and, at each step, appends an element
  // to it and passes it down the chain... so, to boot the
  // engine, we call it with the initial state, an empty list
  (["Nil"]);

// Some tests
const list = ["Cons", 1, ["Cons", 2, ["Cons", 3, ["Nil"]]]];
const print = x => console.log (JSON.stringify (x));

// Sums the list
print( foldr ((x,y) => x+y) (0) (list) );

// Doubles each element
print( map (x => x * 2) (list) );

// Removes even elements
print( filter (x => x % 2) (list) );

// Reverses the list
print( reverse (list) );

// The result of this program is:
// 6
// ["Cons",2,["Cons",4,["Cons",6,["Nil"]]]]
// ["Cons",1,["Cons",2,["Cons",3,["Nil"]]]]
// ["Cons",3,["Cons",2,["Cons",1,["Nil"]]]]
```
