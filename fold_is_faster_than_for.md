## Folds are inherently faster than loops

  I'd call this article "pure languages are inherently faster than imperative languages", but I assumed that would be too provocative and, in general, incorrect. Instead, I'll talk about the specific reason I think this is true: pure folds are inherently faster than imperative loops. If you don't know much about folds, I made a [separate post](https://github.com/MaiaVictor/posts/blob/master/foldr_is_like_replace.md) explaining folds in a hopefully intuitive way.

There is a widespread belief that `loops` are inherently faster than `folds`. For example, an array-processing algorithm implemented with `for` is thought to be faster than the same algorithm implemented with `foldr` (also known as `reduce`) when both are properly optimized. That might be for 3 reasons:

1. Intuitivelly, for-loops are "closer the machine", so they must be faster, right?

2. In most languages where functional styles are available (JavaScript, Python, Ruby, Clojure, Scheme, C# and so on), `foldr` isn't optimized properly; on those cases, `for` is, indeed, faster.

3. In fact, the only mainstream language which optimizes `foldr` properly (Haskell) only does so in some specific cases, which are often overshadowed by the presence of other slow features.

Contrary to popular belief, though, folds are inherently faster than loops. By inherently faster, I mean that any arbitrary program oriented torwards `foldr` will, if properly optimized, always be as fast, and often faster, than the same, properly optimized, program making use of `for`. As such, **impure, imperative languages such as `Rust` and `C` will always be slower than properly implemented pure languages, in the same way that interpreted languages will always be slower than compiled languages**. Sadly, there isn't any pure, performance-oriented mainstream language to serve as an example, so, unlike the compiled vs interpreted debate, this is much more about "what is possible" than "what currently holds". Currently, loops are faster in essentially all mainstream languages, make no mistake. The reason for that, as I will explain, is an extremelly powerful, yet rarely explored, compilation technique known as fusion. Before elaborating on it, though, let's talk about modularity.

### The unspoken cost of modularity

Suppose that you're attempting to write a program that receives a dataset, removes the even elements and returns the double of the rest. I'll use JavaScript because it seems to be the most common language nowadays. This is the naive `for-loop` solution:

```js
// Naive double-of-odds solution
function double_of_odds(array){
  // Start with an empty array
  var result = [];

  // For each element of the array
  for (var i = 0; i < array.length; ++i)

    // If that element is odd
    if (array[i] % 2 === 1)

      // Multiply by 2 and append to the result
      result.push(array[i] * 2);

  // Return the result
  return result;
};
```

That solution, sans micro-optimizations, is optimal, but it is also not modular. Suppose, for example, that you also wanted to write a `double_of_evens` function. In that case, you could copy-paste the function above, but that would result in duplicate code, which is universally avoided. In practice, it is much more common to split the logic in smaller, modular functions, reusing that logic to implement `double_of_evens` and `double_of_odds`:

```js
function filter_odds(array){
  var result = [];
  for (var i = 0; i < array.length; ++i)
    if (i % 2 === 1)
      result.append(i);
  return result;
}
function filter_evens(array){
  var result = [];
  for (var i = 0; i < array.length; ++i)
    if (i % 2 === 0)
      result.append(i);
  return result;
}
function double_all(array){
  var result = [];
  for (var i = 0; i < array.length; ++i)
    result.append(array[i] * 2);
  return result;
};
function double_of_odds(array){
  return double_all(filter_odds(array));
};
function double_of_evens(array){
  return double_all(filter_evens(array));
};
```

The modular approach is more robust, but it is also slower: now `double_of_odds` iterates through the dataset twice, and also allocates an intermediate array (`filter_odds(array)`) that serves for no purpose. This is not good at all: if you just copy-pasted the previous solution, your code would be about **twice as fast**. That sounds like a tradeoff: you either have modularity or performance. So, it begs the question: is it possible to have both?

### Folds allow zero-cost modularity

The answer is: yes, with folds. **Any arbitrarily long chain of transformations (such as `filter_evens`, `double_all` and so on) can always be compressed to a single loop at compile time, as long as each individual transformation is implemented without loops and without recursion!** Now, notice folds can not only be implemented non-recursivelly (see, for example, church-encodings), but they are also universal, which means any loop-based/recursive algorithm over an arbitrary datatype can be expressed as a fold. So, that is exactly what we need! As long as we replace recursion and loops with adequate folds, we can have both maximum modularity and maximum performance.

This is implemented in some functional languages under a technique usually addressed as **fusion** and/or **deforestation**. Fusing any number of consecutive transformations into a single loop makes the compiled binary faster by about a whole integer factor, depending on how modular the original code is. **The same transformation is impossible to apply, generally, to an imperative language that uses loops, thanks to the halting problem.** As such, the only way for an imperative program using `for` to be as fast as a pure program using `foldr`, once properly optimized, is by **completely sacrifying modularity** - i.e., manually writing your entire program logic in a single loop. But how many real-world programs do that?

Think about this carefully: every single program in the world that, for example, iterates through arrays (or any other structure) with loops or recursion will necessarily face a whole integer-factor slowdown as soon as it splits the logic into two or more different functions/objects/components. This is not a 1%, 2% slowdown. This is a 200%, 300%, 400% and so on slowdown - the more modular, the worse. This isn't about Euler Problems, this is about everything: from your Kernel, to your apps, to the web, that slowdown is everywhere. 

Want an actual example? React. The site you're reading right now was probably written using it. React is currently the most popular front-end web-develpment framework. Inside it, there is a `diff` function that iterates through the whole skeleton of your web-page looking for changes. Outside it, there is an application which actually builds that skeleton. Since those are modular (i.e., `diff` and `render` are separate functions), and since both are implemented with loops/recursion, then each site is necessarily processed twice, i.e., `diff(render(...))` isn't fused. If, instead, diff/render were implemented with folds, then every React app could trivially be minified to a JavaScript program that *magically diffs as it renders*, saving the browser from allocating millions of bytes every frame, with no added effort. Moreover, since diff/render is often the main bottleneck of many React apps, countless webpages all around the world could be immediately twice as fast with almost no effort had JavaScript been pure!

### Fusion in action: Haskell vs C

So, you might be asking: if that all is true, why isn't a language such as Haskell enourmously faster than C? And the answer is: (i) again, Haskell doesn't aim for performance above everything; (ii) in a sense, it is. If you restrict yourself to a subset of Haskell that is both total and strict (i.e., avoid recursion, laziness, etc.), then your modular programs will be considerably faster than the same modular programs written in C. **That subset of Haskell is, currently, the fastest programming language in the world for modular programs (i.e., pretty much all real-world programs).**

Observe, for example,  [this C program](https://gist.github.com/MaiaVictor/cd2926877d2477bd7db5129f4582d2ff) and [this Haskell program](https://gist.github.com/MaiaVictor/bfbb47f66dbb4258764b3186d930a1e9). Those are very similar: both alloc an array of 100 billion integers, then increase each element by `64` by calling an `add1` function `64` times, and then output the sum of it all. Note those programs aren't bizarre, constructed edge-cases. They're the dead-simple, ubiquitous pattern of mapping over a dataset. Each separate call of `add1` symbolizes a separate transformation applied to that dataset. To keep things fair, the C program is performing the increments in-place; i.e, `add1` mutates the original array. The Haskell program returns a whole new array each time `add1` is called, so, basically, the Haskell code is actually much worse, because it is supposed to reallocate the 100-billion-element array each time `add1` is called, whereas the C code would only alloc it once. So, which program is faster? Yes, the Haskell one. By a `8x` factor. Mind blown? Not really, when you consider the Haskell one is fused into a single loop.

Note that this program is very simple so I wouldn't be surprised if a "sufficiently smart compiler" (gcc?) could figure it out and fuse the version C to a single loop too, potentially beating Haskell on this demo. But that can not work for the general case, and any minor complication would beat that analysis. Fusion of pure, total programs, on the other side, can always be applied; no need for a "smart" compiler, a dumb one suffices.

Also note this is the kind of thing that won't show up in benchmarks such as [The Computer Benchmark Game](http://benchmarksgame.alioth.debian.org/), because those tests not modular, the loops are all manually fused. The winner is, thus, the compiler with the most micro optimizations. Real-world programs don't look like that, though. They're modular, and modularity costs a lot. 

### Conclusion

So, in short, am I saying that Haskell is much faster than, say, C and Rust, for real-world programs? No! In practice, Haskell uses a lot of recursion and loops, and all it takes is a single one of those to completely break the fusion chain. Moreover, the only structures that are actually fused in Haskell, as far as I know, are `Data.List` and `Data.Vector`. It is also easy to destroy a Haskell program's performance for reasons completely unrelated to folds. Haskell is not a fast functional programming language; it is just the fastest that we managed to create.

That doesn't change the fact that `folds` are inherently faster than recursion and loops. If someone designed an actual functional language oriented torwards performance (i.e., disabling recursion and loops in the same way that Rust disables non-linearity, providing a proper syntax for folds, enabling fusion for all datatypes and not just lists, using linearirty to enable in-place mutation, making use of implicit parallelism whenever possible and disabling garbage-collection), then it would, easily, beat C and Rust by a large margin; not for Computer Language Game Benchmarks, perhaps, but certainly for any real-world program. After all, what is a 20% speedup due to some clever memory-management optimizations when your programs get 2x, 3x, 4x slower whenever you split its logic into separate components?

