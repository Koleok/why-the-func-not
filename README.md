# Why the func not?
practical FP in javascript land

Kyle Chamberlain

2016-12-14

---

## Javascript is already a functional language
It has all of the necessary properties to be considered one, but also accomdates other flavors of programming that are more conventional and comfortable in our current software culture.

This means we have a choice to make, and like water, programmers often choose to take the path of least resistance. In this case that means leaning on the principles we learned in our `java` / `c#` / `ruby` / `etc...` backgrounds.

Javascript adopts concepts from lots of classic functional languages, so lets explore the benefits leveraging some of this awesome power sitting right under our nose everyday.

---

## Null Checks
Lets say an API responds to us with an array of animals: 

```javascript
const animals = [
  { id: 1, name: 'mule' },
  { id: 2, name: 'donkey', friends: [
    { id: 10, name: 'horse' }
  ]},
]
```

We know the following things

- each animal may or may not have a
	- `name` property
	- list of `friends` _(also animals)_
- the first animal in any friend list can be assumed to be the **best friend**.

Now lets say our job is to get an array `[String]` of the names of **best friends** of all our animals.

This is the kind of task we all encounter frequently in the course of working just anywhere in the software universe, I like to call it the **If this is a thing** problem though of course called a null check 😁.

---

1. The **If this is a thing** problem can be approached several ways

	1. Nested for loops that filter out each layer of presence/absence before pushing to a mutable results array

		```javascript
		function bestFriendNames(animals) {  
		  const results = []
		  
		  if (animals.length) {
		    animals.forEach(animal => {
		      if (animal.friends && animal.friends.length) {
		        animal.friends.forEach(friend => {
		          if (friend && friend.name) results.push(friend.name)
		        })
		      }
		    })
		  }
		  
		  return results
		}
		``` 

	1. Unnest and use sequential temp arrays from manual `for` or `forEach` filter operations until the final result is reached.

		```javascript
		function bestFriendsNames(animals) {
		  const friendNames = []
		  
		  // Get the friendNames of any animals with friends
		  if (animals && animals.length) {
		    animals.forEach(animal => {
		      if (animal.friends && animal.friends.length) {
		        friendNames.push(animal.friends[0].name)
		      }
		    })
		  }
		  
		  const result = []
		  
		  // Filter out any undefined values from last step
		  if (friendNames.length) {
		    friendNames.forEach(name => {
		      if (name) result.push(name)
		    })
		  }
		  
		  return result
		} 
		```

	1. Use a `functor` with its generic transformer _(`map`)_, in this case the JS native `Array.map` to pull properties and `Array.filter` to remove falsy values via an identity function `(x => x)` that acts as a predicate and returns false for undefined and null
	
		```javascript
		const bestFriendNames = animals => animals
		  .map(x => x.friends)
		  .filter(x => x)
		  .map(x => x[0].name)
		  .filter(x => x)
		```

### All of these solutions have some things in common
1. each is responsible for reading properties, as well as dealing with `empty`/`null`/`undefined` values.
1. using these approaches our existence checking and filtering basically **must** be implemented _one-off_ and even if we can abstract it, it will always be done _inline_, meaning we can't easily re-use all the effort and thought that goes in to our solving.

> Wouldn't it be great if these functions could be more simple and trusting? Like _Forest Gump_? 

---

> How can we trust all our checking to a dependable abstraction and let the functions do the work we care about?

---

I'll tell you how.

# Maybe _(its a monad)_

Ok I said it, but humor me and lets just think of a monad as a container, it can be constructed with any value, then accept functions to transform that value _(via `map`)_, and also accept functions to put the value in a new/different kind of container _(via `chain`/`flatmap`)_

Every monad has something its good at, a calling if you will. For `Maybe`, that calling is simply to **help us deal differently with values we want and values we don't**

The way does that is simple. Maybe has to represent any value we give it as one of two _"child"_ types, `Just` and `Nothing`. 

When we ask our maybe to transform its value _(`map`)_, or to transform the value and put it in a new container _(`chain`)_, it will just skip over our request if the value it contains is considered a `Nothing` value. 
_ex._

```javascript
const add1 = x => x + 1 

Just(1).map(add1).map(add1)    //=> Just(3)
Nothing(1).map(add1).map(add1) //=> Nothing()
```

---

`Maybe`, and Monads in general are just an idea, so the actual implementation we use is irrelevant, we could make our own. However the idea here is to do **LESS** work, so lets use someone else's and benefit from all the time they put into it, I like the `Maybe` from [sanctuaryJs](https://sanctuary.js.org/#Maybe).

Using some other utilities from [ramda](http://ramdajs.com/) and [sanctuaryJs](https://sanctuary.js.org) _(my standard FP javascript toolbelt)_, we can solve the problem from above in a cool new way.

First a little glossary, rather than talk about these individually I have linked to the documentation line for each.

- Ramda
	- [`pipe`](http://ramdajs.com/docs/#pipe)
	- [`pipeK`](http://ramdajs.com/docs/#pipeK)
	- [`map`](http://ramdajs.com/docs/#map) 
- Sanctuary
	- [`Maybe.of`](https://github.com/sanctuary-js/sanctuary#maybeof--a---maybea) 
	- [`justs`](https://github.com/sanctuary-js/sanctuary#justs) 
	- [`head`](https://github.com/sanctuary-js/sanctuary#head)
	- [`get`](https://github.com/sanctuary-js/sanctuary#get)  

	
Now we can:

```javascript
import R from 'ramda';
import S from 'sanctuary';

const bestFriendsName = R.pipeK(
  S.get(Array, 'friends'),  // <-- return Just(value of 'friends' prop)
  S.head,				    // <-- return Just(first element of array) 
  S.get(String, 'name')     // <-- return Just(value of 'name' prop)
)

const bestFriendNames = R.pipe(
  R.map(S.Maybe.of),		// <-- turn each element in array into a Just
  R.map(firstFriendName),
  S.justs                   // <-- pull only the Just values out of monad and ignore Nothings
)
```
