# Monadic Onions

## Pointy Functor Factory

Before we go any further, I have a confession to make: I haven't been fully honest about that `of` method we've placed on each of our types. Turns out, it is not there to avoid the `new` keyword, but rather to place values in what's called a *default minimal context*. Yes, `of` does not actually take the place of a constructor - it is part of an important interface we call *Pointed*.

> A *pointed functor* is a functor with an `of` method

What's important here is the ability to drop any value in our type and start mapping away.

```js
IO.of('tetris').map(concat(' master'));
// IO('tetris master')

Maybe.of(1336).map(add(1));
// Maybe(1337)

Task.of({
  id: 2,
}).map(_.prop('id'));
// Task(2)

Either.of('The past, present and future walk into a bar...').map(
  concat('it was tense.')
);
// Right('The past, present and future walk into a bar...it was tense.')
```

If you recall, `IO` and `Task`'s constructors expect a function as their argument, but `Maybe` and `Either` do not. The motivation for this interface is a common, consistent way to place a value into our functor without the complexities and specific demands of constructors. The term "default minimal context" lacks precision, yet captures the idea well: we'd like to lift any value in our type and `map` away per usual with the expected behaviour of whichever functor.

One important correction I must make at this point, pun intended, is that `Left.of` doesn't make any sense. Each functor must have one way to place a value inside it and with `Either`, that's `new Right(x)`. We define `of` using `Right` because if our type *can* `map`, it *should* `map`. Looking at the examples above, we should have an intuition about how `of` will usually work and `Left` breaks that mold.

You may have heard of functions such as `pure`, `point`, `unit`, and `return`. These are various monikers for our `of` method, international function of mystery. `of` will become important when we start using monads because, as we will see, it's our responsibility to place values back into the type manually.

To avoid the `new` keyword, there are several standard JavaScript tricks or libraries so let's use them and use `of` like a responsible adult from here on out. I recommend using functor instances from `folktale`, `ramda` or `fantasy-land` as they provide the correct `of` method as well as nice constructors that don't rely on `new`.


## Mixing Metaphors

<img src="images/onion.png" alt="http://www.organicchemistry.com/wp-content/uploads/BPOCchapter6-6htm-41.png" />

You see, in addition to space burritos (if you've heard the rumors), monads are like onions. Allow me to demonstrate with a common situation:

```js
// Support
// ===========================
var fs = require('fs');

//  readFile :: String -> IO String
var readFile = function(filename) {
  return new IO(function() {
    return fs.readFileSync(filename, 'utf-8');
  });
};

//  print :: String -> IO String
var print = function(x) {
  return new IO(function() {
    console.log(x);
    return x;
  });
};

// Example
// ===========================
//  cat :: String -> IO (IO String)
var cat = compose(map(print), readFile);

cat('.git/config');
// IO(IO('[core]\nrepositoryformatversion = 0\n'))
```

What we've got here is an `IO` trapped inside another `IO` because `print` introduced a second `IO` during our `map`. To continue working with our string, we must `map(map(f))` and to observe the effect, we must `unsafePerformIO().unsafePerformIO()`.

```js
//  cat :: String -> IO (IO String)
var cat = compose(map(print), readFile);

//  catFirstChar :: String -> IO (IO String)
var catFirstChar = compose(map(map(head)), cat);

catFirstChar(".git/config");
// IO(IO("["))
```

While it is nice to see that we have two effects packaged up and ready to go in our application, it feels a bit like working in two hazmat suits and we end up with an uncomfortably awkward API. Let's look at another situation:

```js
//  safeProp :: Key -> {Key: a} -> Maybe a
var safeProp = curry(function(x, obj) {
  return new Maybe(obj[x]);
});

//  safeHead :: [a] -> Maybe a
var safeHead = safeProp(0);

//  firstAddressStreet :: User -> Maybe (Maybe (Maybe Street) )
var firstAddressStreet = compose(
  map(map(safeProp('street'))), map(safeHead), safeProp('addresses')
);

firstAddressStreet({
  addresses: [{
    street: {
      name: 'Mulburry',
      number: 8402,
    },
    postcode: 'WC2N',
  }],
});
// Maybe(Maybe(Maybe({name: 'Mulburry', number: 8402})))
```

Again, we see this nested functor situation where it's neat to see there are three possible failures in our function, but it's a little presumptuous to expect a caller to `map` three times to get at the value - we'd only just met. This pattern will arise time and time again and it is the primary situation where we'll need to shine the mighty monad symbol into the night sky.

I said monads are like onions because tears well up as we peel back layer of the nested functor with `map` to get at the inner value. We can dry our eyes, take a deep breath, and use a method called `join`.

```js
var mmo = Maybe.of(Maybe.of('nunchucks'));
// Maybe(Maybe('nunchucks'))

mmo.join();
// Maybe('nunchucks')

var ioio = IO.of(IO.of('pizza'));
// IO(IO('pizza'))

ioio.join();
// IO('pizza')

var ttt = Task.of(Task.of(Task.of('sewers')));
// Task(Task(Task('sewers')));

ttt.join();
// Task(Task('sewers'))
```

If we have two layers of the same type, we can smash them together with `join`. This ability to join together, this functor matrimony, is what makes a monad a monad. Let's inch toward the full definition with something a little more accurate:

> Monads are pointed functors that can flatten

Any functor which defines a `join` method, has an `of` method, and obeys a few laws is a monad. Defining `join` is not too difficult so let's do so for `Maybe`:

```js
Maybe.prototype.join = function() {
  return this.isNothing() ? Maybe.of(null) : this.__value;
}
```

There, simple as consuming one's twin in the womb. If we have a `Maybe(Maybe(x))` then `.__value` will just remove the unnecessary extra layer and we can safely `map` from there. Otherwise, we'll just have the one `Maybe` as nothing would have been mapped in the first place.

Now that we have a `join` method, let's sprinkle some magic monad dust over the `firstAddressStreet` example and see it in action:

```js
//  join :: Monad m => m (m a) -> m a
var join = function(mma) {
  return mma.join();
};

//  firstAddressStreet :: User -> Maybe Street
var firstAddressStreet = compose(
  join, map(safeProp('street')), join, map(safeHead), safeProp('addresses')
);

firstAddressStreet({
  addresses: [{
    street: {
      name: 'Mulburry',
      number: 8402,
    },
    postcode: 'WC2N',
  }],
});
// Maybe({name: 'Mulburry', number: 8402})
```

We added `join` wherever we encountered the nested `Maybe`'s to keep them from getting out of hand. Let's do the same with `IO` to give us a feel for that.

```js
IO.prototype.join = function() {
  return this.unsafePerformIO();
}
```

Again, we simply remove one layer. Mind you, we have not thrown out purity, but merely removed one layer of excess shrink wrap.

```js
//  log :: a -> IO a
var log = function(x) {
  return new IO(function() {
    console.log(x);
    return x;
  });
};

//  setStyle :: Selector -> CSSProps -> IO DOM
var setStyle = curry(function(sel, props) {
  return new IO(function() {
    return jQuery(sel).css(props);
  });
});

//  getItem :: String -> IO String
var getItem = function(key) {
  return new IO(function() {
    return localStorage.getItem(key);
  });
};

//  applyPreferences :: String -> IO DOM
var applyPreferences = compose(
  join, map(setStyle('#main')), join, map(log), map(JSON.parse), getItem
);


applyPreferences('preferences').unsafePerformIO();
// Object {backgroundColor: "green"}
// <div style="background-color: 'green'"/>
```

`getItem` returns an `IO String` so we `map` to parse it. Both `log` and `setStyle` return `IO`'s themselves so we must `join` to keep our nesting under control.

## My chain hits my chest

<img src="images/chain.jpg" alt="chain" />

You might have noticed a pattern. We often end up calling `join` right after a `map`. Let's abstract this into a function called `chain`.

```js
//  chain :: Monad m => (a -> m b) -> m a -> m b
var chain = curry(function(f, m){
  return m.map(f).join(); // or compose(join, map(f))(m)
});
```

We'll just bundle up this map/join combo into a single function. If you've read about monads previously, you might have seen `chain` called `>>=` (pronounced bind) or `flatMap` which are all aliases for the same concept. I personally think `flatMap` is the most accurate name, but we'll stick with `chain` as it's the widely accepted name in JS. Let's refactor the two examples above with `chain`:

```js
// map/join
var firstAddressStreet = compose(
  join, map(safeProp('street')), join, map(safeHead), safeProp('addresses')
);

// chain
var firstAddressStreet = compose(
  chain(safeProp('street')), chain(safeHead), safeProp('addresses')
);



// map/join
var applyPreferences = compose(
  join, map(setStyle('#main')), join, map(log), map(JSON.parse), getItem
);

// chain
var applyPreferences = compose(
  chain(setStyle('#main')), chain(log), map(JSON.parse), getItem
);
```

I swapped out any `map/join` with our new `chain` function to tidy things up a bit. Cleanliness is nice and all, but there's more to `chain` than meets the eye - it's more of tornado than a vacuum. Because `chain` effortlessly nests effects, we can capture both *sequence* and *variable assignment* in a purely functional way.

```js
// getJSON :: Url -> Params -> Task JSON
// querySelector :: Selector -> IO DOM


getJSON('/authenticate', {
    username: 'stale',
    password: 'crackers',
  })
  .chain(function(user) {
    return getJSON('/friends', {
      user_id: user.id,
    });
  });
// Task([{name: 'Seimith', id: 14}, {name: 'Ric', id: 39}]);


querySelector('input.username').chain(function(uname) {
  return querySelector('input.email').chain(function(email) {
    return IO.of(
      'Welcome ' + uname.value + ' ' + 'prepare for spam at ' + email.value
    );
  });
});
// IO('Welcome Olivia prepare for spam at olivia@tremorcontrol.net');


Maybe.of(3).chain(function(three) {
  return Maybe.of(2).map(add(three));
});
// Maybe(5);


Maybe.of(null).chain(safeProp('address')).chain(safeProp('street'));
// Maybe(null);
```

We could have written these examples with `compose`, but we'd need a few helper functions and this style lends itself to explicit variable assignment via closure anyhow. Instead we're using the infix version of `chain` which, incidentally, can be derived from `map` and `join` for any type automatically: `t.prototype.chain = function(f) { return this.map(f).join(); }`. We can also define `chain` manually if we'd like a false sense of performance, though we must take care to maintain the correct functionality - that is, it must equal `map` followed by `join`. An interesting fact is that we can derive `map` for free if we've created `chain` simply by bottling the value back up when we're finished with `of`. With `chain`, we can also define `join` as `chain(id)`. It may feel like playing Texas Hold em' with a rhinestone magician in that I'm just pulling things out of my behind, but, as with most mathematics, all of these principled constructs are interrelated. Lots of these derivations are mentioned in the [fantasyland](https://github.com/fantasyland/fantasy-land) repo, which is the official specification for algebraic data types in JavaScript.

Anyways, let's get to the examples above. In the first example, we see two `Task`'s chained in a sequence of asynchronous actions - first it retrieves the `user`, then it finds the friends with that user's id. We use `chain` to avoid a `Task(Task([Friend]))` situation.

Next, we use `querySelector` to find a few different inputs and create a welcoming message. Notice how we have access to both `uname` and `email` at the innermost function - this is functional variable assignment at its finest. Since `IO` is graciously lending us its value, we are in charge of putting it back how we found it - we wouldn't want to break its trust (and our program). `IO.of` is the perfect tool for the job and it's why Pointed is an important prerequisite to the Monad interface. However, we could choose to `map` as that would also return the correct type:

```js
querySelector('input.username').chain(function(uname) {
  return querySelector('input.email').map(function(email) {
    return 'Welcome ' + uname.value + ' prepare for spam at ' + email.value;
  });
});
// IO('Welcome Olivia prepare for spam at olivia@tremorcontrol.net');
```

Finally, we have two examples using `Maybe`. Since `chain` is mapping under the hood, if any value is `null`, we stop the computation dead in its tracks.

Don't worry if these examples are hard to grasp at first. Play with them. Poke them with a stick. Smash them to bits and reassemble. Remember to `map` when returning a "normal" value and `chain` when we're returning another functor.

As a reminder, this does not work with two different nested types. Functor composition and later, monad transformers, can help us in that situation.

#Power trip

Container style programming can be confusing at times. We sometimes find ourselves struggling to understand how many containers deep a value is or if we need `map` or `chain` (soon we'll see more container methods). We can greatly improve debugging with tricks like implementing `inspect` and we'll learn how to create a "stack" that can handle whatever effects we throw at it, but there are times when we question if it's worth the hassle.

I'd like to swing the fiery monadic sword for a moment to exhibit the power of programming this way.

Let's read a file, then upload it directly afterward:

```js
// readFile :: Filename -> Either String (Task Error String)
// httpPost :: String -> Task Error JSON

//  upload :: String -> Either String (Task Error JSON)
var upload = compose(map(chain(httpPost('/uploads'))), readFile);
```

Here, we are branching our code several times. Looking at the type signatures I can see that we protect against 3 errors - `readFile` uses `Either` to validate the input (perhaps ensuring the filename is present), `readFile` may error when accessing the file as expressed in the first type parameter of `Task`, and the upload may fail for whatever reason which is expressed by the `Error` in `httpPost`. We casually pull off two nested, sequential asynchronous actions with `chain`.

All of this is achieved in one linear left to right flow. This is all pure and declarative. It holds equational reasoning and reliable properties. We aren't forced to add needless and confusing variable names. Our `upload` function is written against generic interfaces and not specific one-off APIs. It's one bloody line for goodness sake.

For contrast, let's look at the standard imperative way to pull this off:

```js
//  upload :: String -> (String -> a) -> Void
var upload = function(filename, callback) {
  if (!filename) {
    throw "You need a filename!";
  } else {
    readFile(filename, function(err, contents) {
      if (err) throw err;
      httpPost(contents, function(err, json) {
        if (err) throw err;
        callback(json);
      });
    });
  }
};
```

Well isn't that the devil's arithmetic. We're pinballed through a volatile maze of madness. Imagine if it were a typical app that also mutated variables along the way! We'd be in the tar pit indeed.

#Theory

The first law we'll look at is associativity, but perhaps not in the way you're used to it.

```js
// associativity
compose(join, map(join)) == compose(join, join);
```

These laws get at the nested nature of monads so associativity focuses on joining the inner or outer types first to achieve the same result. A picture might be more instructive:

<img src="images/monad_associativity.png" alt="monad associativity law" />

Starting with the top left moving downward, we can `join` the outer two `M`'s of `M(M(M a))` first then cruise over to our desired `M a` with another `join`. Alternatively, we can pop the hood and flatten the inner two `M`'s with `map(join)`. We end up with the same `M a` regardless of if we join the inner or outer `M`'s first and that's what associativity is all about. It's worth noting that `map(join) != join`. The intermediate steps can vary in value, but the end result of the last `join` will be the same.

The second law is similar:

```js
// identity for all (M a)
compose(join, of) === compose(join, map(of)) === id
```

It states that, for any monad `M`, `of` and `join` amounts to `id`. We can also `map(of)` and attack it from the inside out. We call this "triangle identity" because it makes such a shape when visualized:

<img src="images/triangle_identity.png" alt="monad identity law" />

If we start at the top left heading right, we can see that `of` does indeed drop our `M a` in another `M` container. Then if we move downward and `join` it, we get the same as if we just called `id` in the first place. Moving right to left, we see that if we sneak under the covers with `map` and call `of` of the plain `a`, we'll still end up with `M (M a)` and `join`ing will bring us back to square one.

I should mention that I've just written `of`, however, it must be the specific `M.of` for whatever monad we're using.

Now, I've seen these laws, identity and associativity, somewhere before... Hold on, I'm thinking...Yes of course! They are the laws for a category. But that would mean we need a composition function to complete the definition. Behold:

```js
var mcompose = function(f, g) {
  return compose(chain(f), chain(g));
};

// left identity
mcompose(M, f) == f;

// right identity
mcompose(f, M) == f;

// associativity
mcompose(mcompose(f, g), h) === mcompose(f, mcompose(g, h));
```

They are the category laws after all. Monads form a category called the "Kleisli category" where all objects are monads and morphisms are chained functions. I don't mean to taunt you with bits and bobs of category theory without much explanation of how the jigsaw fits together. The intention is to scratch the surface enough to show the relevance and spark some interest while focusing on the practical properties we can use each day.


## In Summary

Monads let us drill downward into nested computations. We can assign variables, run sequential effects, perform asynchronous tasks, all without laying one brick in a pyramid of doom. They come to the rescue when a value finds itself jailed in multiple layers of the same type. With the help of the trusty sidekick "pointed", monads are able to lend us an unboxed value and know we'll be able to place it back in when we're done.

Yes, monads are very powerful, yet we still find ourselves needing some extra container functions. For instance, what if we wanted to run a list of api calls at once, then gather the results? We can accomplish this task with monads, but we'd have to wait for each one to finish before calling the next. What about combining several validations? We'd like to continue validating to gather the list of errors, but monads would stop the show after the first `Left` entered the picture.

In the next chapter, we'll see how applicative functors fit into the container world and why we prefer them to monads in many cases.

[Chapter 10: Applicative Functors](ch10.md)


## Exercises

```js
// Exercise 1
// ==========
// Use safeProp and map/join or chain to safely get the street name when given
// a user.

var safeProp = _.curry(function(x, o) {
  return Maybe.of(o[x]);
});
var user = {
  id: 2,
  name: 'albert',
  address: {
    street: {
      number: 22,
      name: 'Walnut St',
    },
  },
};

var ex1 = undefined;


// Exercise 2
// ==========
// Use getFile to get the filename, remove the directory so it's just the file,
// then purely log it.

var getFile = function() {
  return new IO(function() {
    return __filename;
  });
};

var pureLog = function(x) {
  return new IO(function() {
    console.log(x);
    return 'logged ' + x;
  });
};

var ex2 = undefined;



// Exercise 3
// ==========
// Use getPost() then pass the post's id to getComments().
//
var getPost = function(i) {
  return new Task(function(rej, res) {
    setTimeout(function() {
      res({
        id: i,
        title: 'Love them tasks',
      });
    }, 300);
  });
};

var getComments = function(i) {
  return new Task(function(rej, res) {
    setTimeout(function() {
      res([{
        post_id: i,
        body: 'This book should be illegal',
      }, {
        post_id: i,
        body: 'Monads are like smelly shallots',
      }]);
    }, 300);
  });
};


var ex3 = undefined;


// Exercise 4
// ==========
// Use validateEmail, addToMailingList, and emailBlast to implement ex4's type
// signature.

//  addToMailingList :: Email -> IO([Email])
var addToMailingList = (function(list) {
  return function(email) {
    return new IO(function() {
      list.push(email);
      return list;
    });
  };
})([]);

function emailBlast(list) {
  return new IO(function() {
    return 'emailed: ' + list.join(',');
  });
}

var validateEmail = function(x) {
  return x.match(/\S+@\S+\.\S+/) ? (new Right(x)) : (new Left('invalid email'));
};

//  ex4 :: Email -> Either String (IO String)
var ex4 = undefined;
```
