---
title: "Using Haskell's Freedom in JavaScript"
date: 2021-08-02T20:31:58+02:00
draft: false
---

One of the topics mention when asking "why learn haskell?" is the claim that it can make you a better
programmer in other languages as well. This is not an outrageous claim by any means as learning Java 
for instance will teach you OO and therefore you'll know how to use OO in JavaScript.

That said. You do not need to learn Haskell to understand what `map` does in JS any more than you need C to know what
a for loop does in JS. And while JS does facilitate OO, making the claim that Java could enrich your JS experience prominent. 
JS does not have any monads or kinds, not even types really. So the claim really boils down to 
"does the techniques used in Haskell translate to JS programming?". With this post I wanted to answer that and 
also see if I could learn a bit more of how the simple, but esoteric concepts of fix points and free monads work. 
Programming those two thing in JS will be a test for whether I understand the Haskell implementation or not. 

## Straight Forward Implementations 
Lets say you need a simple evaluator for a simple algebra, with only plus, multiply and a concept of numbers. I will show a simple
solution to this in both Haskell and JavaScript. Then we will see how we can abstract and compartmentalize this process using
both the fix data structure and the free monad.

### Haskell
We start by creating a datatype to represent the expressions. This is the abstract syntax tree for our evaluator so will call it
AST for short. 
```haskell
data AST = Value Int
         | Add AST AST
         | Mul AST AST
         deriving Show
```
This says that the data type `AST` is either a `Value`, `Add` or `Mul`.
The constructor `Value` holds an integer, this is the numbers in our algebra. The two other constructors, `Add`
and `Mul` both takes to arguments of `AST` so it's a recursive data structure. 

An evaluator function in Haskell might look like this:
```haskell
eval :: AST -> Int                      -- 1.
eval (Value n)   = n                    -- 2.
eval (Add e1 e2) = eval e1 + eval e2    -- 3.
eval (Mul e1 e2) = eval e1 * eval e2    -- 3.
```
1. This is the type signature. It says that eval is a function from our abstract syntax tree to an integer.
2. Since we have three constructors in `AST` we naturally have three cases in `eval`. This is the first, and also the easiest.
It says that `eval` applied to the value `Value n` is equal to `n`, or for imperative programers, given `Value n`, return `n`.
3. These cases are more complex, but also straight forward. Given `Add e1 e2` eval both `e1` and `e2`, and add the answers. The same goes for `Mul` but multiply instead.

Now for an example lets say we have an instance of `AST`:
```haskell
expr :: AST
expr = Add
        (Value 2)
        (Mul
            (Value 3)
            (Value 5))
-- expr = 2 + (3 * 5)
```

applying `eval` to `expr` yields:
```haskell
eval expr -- 17
```

Now lets see this in JS.

### JavaScript
First up, the expression. JS does not have type declaration like Haskell does. 
In JS we instead just jump straight to making a instance of an AST. 
```javascript
expr = {
    type: "add",
    left: {
        type: "value",
        value: 2
    },
    right: {
        type: "mul",
        left: {
            type: "value",
            value: 3
        },
        right: {
            type: "value",
            value: 5
        }
    }
}  
// expr = 2 + (3 * 5)
```
In JS we use objects to handle the task of representing data. We use the `type` key to note the different constructors, and `left`, 
`right` and `value` will keep track of the data.

An evaluator for this in JS might look like:
```javascript
function eval(ast) {
    if (ast.type === "value") {                     // 1.
        return ast.value;
    }

    if (ast.type === "mul") {                       // 2.
        return eval(ast.left) * eval(ast.right);
    } else if (ast.type === "add") {                // 2.
        return eval(ast.left) + eval(ast.right);
    }
}

eval(expr) // 17
```

1. If the type of `ast` is a value, simply return that value.
2. Like in Haskell we add or multiply the two branches depending on the value of `type`.
---
**NOTE**

##### This is unsafe as JS is a dynamic language and the value `ast` could always be mistyped. In this post we will continue to ignore this, so that the concepts we're learning are more clear.

---

Not that we know what we are making, lets se if we could decouple the recursive aspect and the algebraic aspect of the evaluator. 

## Fix
A fix point in math is a value `x` where a function `f(x) = x`. Basically the function returns the same value. We have these 
functions in programming as well. We can even represent the concept, 
which is what we seek to do. 

## Fix in Haskell
We can represent a fix point data structure in Haskell like this:
```haskell
newtype Fix f = Fix { unFix :: f (Fix f) }
```
This is a bit more complicated then `AST`. We pack some things into the type declaration. I'll explain them to the 
non-Haskellers. First of the `newtype` is like `data` but allows for more efficient representations, it only works when we
only have one constructor. Since `Fix` can be sees as container, we might need a way to get data from that container. 
This is something that Haskell can automate for us, so instead for writing something similar to the first case in `eval`, e.g: 
`eval (Value n) = n`, we instead let Haskell do that by declaring the `unFix` function in the data declaration for `Fix`. 
So any application of `unFix` onto a value of `Fix` gives us the value `f (Fix f)`.

Now for both Haskellers and others. The type `f (Fix f)` is kinda esoteric don't you think? But there is a some information we can 
hold on to, so as to guide our understanding. First notice that `f` must be a type constructor which accepts a parameter. In the
context of `Fix` we fill that parameter with `(Fix f)`, a new iteration. This is what allows our `Fix` data type to act
like a fix point in math. We can extract a new `Fix` context from the information in any other `Fix` context! The reason that the `f`
is there is so we can fill our `Fix` context with more information, for example an algebra. You might see where this is heading. 
We have a data structure which hold information of how to recurse, and can be filled with other info in between the recursive points.

This is what we wanted when we sought to decouple the recursion and algebra of `eval`!

### Using Fix
So, what can we put in place of `f`? Well lets say we have a different version of `AST`, one which is no longer recursive.
```haskell
data AST r = Value Int
           | Add r r
           | Mul r r
           deriving Show
```
Looks rather familiar, but the `r` is new, and it replaced `AST` in `Add` and `Mul`. So we created a more flexible version of `AST`.
This means that `AST` now has a type parameter, and therefore fits in `Fix f` as a replacement for `f`. 

We can now make an expression using fix:
```haskell
exprFix :: Fix AST                          -- 1.
exprFix = Fix (Add                          -- 2.
                (Fix (Value 2))             
                (Fix (Mul 
                        (Fix (Value 3)) 
                        (Fix (Value 5)))))
```
1. The type signatur tells us that `AST` took the place of `f` in `Fix f`. Likewise `Fix AST` takes the place of `r` in `AST`, although you can't see it her it follows from the definition of `Fix`.
2. Her we see that we build the data structure using `Fix` for recursion and `AST` to hold values. Notice the `Fix,AST,Fix,AST` pattern, toggling between the to each time. Also see that since `Value` does not contain any `Fix`, it terminates the recursion. Therefore the data structure need not be infinite.

### Folding fix
So we have this data type, but we still need to use it somehow. It's intent is to represent recursion, so a neat thing would be
to be able to fold it, get a value from the values in the structure. This function is from the Haskell implementation of fix, and 
it can be a bit tricky.
```haskell
foldFix :: Functor h => (h a -> a) -> Fix a -> a    -- 1.
foldFix f = go where go = f . fmap go . unFix       -- 2.
```

1. This type signatur is more involved than previously seen in this post. The first part says that we need `h` to be a functor. 
In our example the functor will be `AST`, we will see later that it's trivial to make `AST` a functor. Next we take a function from
a functor `h a` to the value it holds `a`. Repeatedly applying this function to the `Fix a` structure will yield an `a`.

2. Now lets look at this `go` function. It first un fixes the fix structure, this gives us the expression `f (Fix f)`. 
Then it maps the function `go` (itself!) onto the result from fix?!, but that means we recurse forever, right? `f` is never called.
No, the function in `fmap` may not always be used, so if the value in `f` is not one where `go` is used, then the recursion stops!
And after that we apply `f`. The trick then is that the values where `go` is unused is the one where `a` is contained.

Now lets see that functor implementation for `AST`.

### Functor 
To make something a functor it need to implement fmap, thats it! Well the implementation should not alter the structure, but thats
pretty simple to accomplish. Basically, `Value` becomes `Value`, `Add` becomes `Add`, and you can guess what `Mul` becomes.
```haskell
instance Functor AST where
    fmap _ (Value n) = Value n
    fmap f (Add a b) = Add (f a) (f b)
    fmap f (Mul a b) = Mul (f a) (f b)
```
Its really a very simple functor. Just apply the function to the instances of `r`.


### Algebra
Now that all the components to make fix work is in place, we can look at the algebra. See how simple it is? This is the reward!
```haskell
algebra :: AST Int -> Int       -- 1.
algebra (Value n)   = n         -- 2.
algebra (Add a1 a2) = a1 + a2   -- 3.
algebra (Mul a1 a2) = a1 * a2   -- 3.
```

1. Notice that this time our evaluation goes from an `AST Int` to `Int`. That `Int` is important!
2. The `Value n` case is once again dead simple, its just equal to `n`.
3. But this time those other cases are also simple. Think about it, they merly says that adding or multiplying to numbers is the
same as adding or multiplying two numbers.


### Putting it Together
The last bit is writing the eval function, this is surprisingly easy. 
```haskell
evalFix :: Fix AST -> Int
evalFix = foldFix algebra  


evalFix exprFix -- 17
```
There that was all. Just fold over a `Fix AST` using `algebra`. Lets try to copy this in JS.

## Fix in JavaScript
Lets start this off by once again looking at the expression.
```javascript
exprFix = {
    type: "fix",
    item: {
        type: "add",
        left: {
            type: "fix",
            item: {
                type: "value",
                value: 2
            }
        },
        right: {
            type: "fix",
            item: {
                type: "mul",
                left: {
                    type: "fix",
                    item: {
                        type: "value",
                        value: 3
                    }
                },
                right: {
                    type: "fix",
                    item: {
                        type: "value",
                        value: 5
                    }
                }
            }
        }
    }
}
```
Well this is a lot more bloated than the first expression. But the gain will be the same in JS as in Haskell. This is really two
data structures in one. One of which act as recursive points, the other holds the actual values.

We need an `unFix` function her as well, so to make things more obvious later. Well also define a map function now that it's usage
is known.
```javascript
function unFix(fix) {
    return fix.item;
}

function mapExpr(fn, ast) {
    if (ast.type !== "value") {
        ast.left = fn(ast.left);
        ast.right = fn(ast.right);
    }

    return ast;  
}
```
### Folding in JavaScript
Now lets see if we can define the `foldFix` function. I came up with this:
```javascript
function foldFix(f, fix) {
    const go = (iter) => {
        return f(mapExpr(go, unFix(iter)));
    };
    return go(fix);
}
```
This is the same technique used in Haskell. It works just as fine in JS. `foldFix` accepts two arguments, one is the function that
does the "work" and the other is the fix data structure. Inside `foldFix` we define a function `go` that takes care of the recursing.
First by unfixing the fix point, then mapping itself onto the result, lastly calling the working function on the result. 

This made it more apparent to me why this technique works, and why the type signature of `algebra` was `AST Int -> Int`. The `go`
function first "digs" itself into the structure, and then phi is applied on the way out again. Therefore it's guarantied that when
the `phi` function is applied, the content in the structure at that point is already folded. 

### Eval
The rest of the code is pretty straight forward so I'll just show you.
```javascript
function algebra(ast) {
    switch (ast.type) {
        case "value":
            return ast.value;
        case "add":
            return ast.left + ast.right;
        case "mul":
            return ast.left * ast.right;
    }
}

function evalFix(ast) {
    return foldFix(algebra, ast);
}

evalFix(exprFix) // 17
```

## Free
Fix is great an all, but that `Value` type was a nuisance. It didn't really need to be fixed, but because of how `Fix` was 
specified we needed to put it in a `Fix` constructor in order to fit it into the data structure. This aided in bloating the 
expression, but it turnes out that solving this problem gives us more power as well.

You see, we can solve it by giving `Fix` another constructor. This acts like `Value`. If we do this we will have 
created whats known as the free monad. What this means is that we can make a structure much like fix, and define functor, applicative
and monad type classes for this structure. Then we can intervene another data structure like `AST` and automatically have
those type classes defined for the new intervened structure! 

Lets get to it.

## Free in Haskell
The free monad is gonna look familiar.
```haskell
data Free f a = Pure a | Free (f (Free f a))
```
Seen something like this before? 

The `a` is new, and so is `Pure a`. We already discussed how this will help us so lets not repeat ourself.

Since defining this as a functor, applicative or monad isn't gonna help us reach our goal I'll just leave it as an exercise. In the links section of this post there is a link to all code on my github. There you will find the answers.

### Using Fix
So lets redefine `AST` a final time. 
```haskell
data AST r = Add r r 
           | Mul r r 

instance Functor AST where
    fmap f (Add a b) = Add (f a) (f b)
    fmap f (Mul a b) = Mul (f a) (f b)
```
Nothing is new here, rather something is missing. 

### Folding Free
One neat thing about the `Free` is that, in my opinion, its fold function is clearer than `Fix`. 
```haskell
foldFree :: Functor f => (f a -> a) -> Free f a -> a    -- 1.
foldFree _   (Pure a) = a                               -- 2. 
foldFree phi (Free m) = phi (fmap (foldFree phi) m)     -- 3.
```

1. Pretty much the same as fix, although its more obvious where the `a` value originates.
2. This one is dead simple, if the value is `Pure a` return `a`. This also makes the termination obvious.
3. Same as with fix. Instead of using a function, i.e. `unFix`, to deconstruct the data structure, we instead use pattern matching.

### Finally Using Free
Now lets create an expression using `Free`.
```haskell
exprFree :: Free AST Int
exprFree = Free (Add
                    (Pure 2)
                    (Free (Mul
                            (Pure 3)
                            (Pure 5))))
```
More involved than `expr`, but less so than `exprFix`. Writing the `algebra` function is the same, but we can now drop the `Value` case.
```haskell
algebra :: AST Int -> Int
algebra (Add a b) = a + b
algebra (Mul a b) = a * b
```
Lastly the eval function is just as easy.
```haskell
evalFree :: Free AST Int -> Int 
evalFree = foldFree algebra

evalFree exprFree -- 17
```

## Free in JavaScript
Now lets see what this does for the JS implementation. 

Like previously we start with the expression. 
```javascript
exprFree = {
    type: "free",
    item: {
        type: "add",
        left: {
            type: "pure",
            item: 2
        },
        right: {
            type: "free",
            item: {
                type: "mul",
                left: {
                    type: "pure",
                    item: 3
                },
                right: {
                    type: "pure",
                    item: 5
                }
            }
        }
    }
}
```
As you could imagine it's more succinct. Now for the rest, we can be a bit cheeky her, since we have two function that already works for AST.
```javascript
/* copy mapExpr and algebra from fix. JS is dynamic so they still work */

function unFree(free) {
    return free.item;
}

function foldFree(phi, free) {
    if (free.type === "pure") {
        return unFree(free);
    }
    // otherwise free.type === "free"
    return phi(mapExpr(x => foldFree(phi, x), unFree(free)));
}
```
I defined `unFree` mostly for clarity. There is no `go` function here, instead `foldFree` is the recursive function. This is 
also what we did in Haskell. If we get a pure value we terminate and return the item in pure, otherwise we do the same type of recursion we did for fix. 

The rest is the same, just like with Haskell.
```javascript
function evalFree(ast) {
    return foldFree(algebra, ast);
}

evalFree(exprFree) // 17
```

## Conclusion
We see that both those concepts translates to JS, with out much stretch as well. Both does something for DRY programming, although
they might not have the same yield in JS as in Haskell. 

Still seeing a very Haskell-ish concept translate so easily to another language is neat, I think learning more Haskell is
gonna be more valuable for the JS programmer in me.

I recommend trying something similar yourself.

## Links
[Chris Penners article was the inspiration for this post](https://chrispenner.ca/posts/asts-with-fix-and-free)  
[Wikibooks chapter on fix](https://en.wikibooks.org/wiki/Haskell/Fix_and_recursion)  
[Stackoverflow thread on free](https://stackoverflow.com/questions/13352205/what-are-free-monads)  
[Wikipedia on fixed points](https://en.wikipedia.org/wiki/Fixed_point_%28mathematics%29)