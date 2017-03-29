---
layout: post
title:  "A Promise is a Promise"
date:   2017-03-29 16:08:07 +0000
---


A JavaScript Promise is an _object_ that represents the eventual result of an asynchronous operation. This means you won't have immediate access to the result, rather you have to wait until the Promise has been settled. It will return either a resolved value, or a reason why it couldn't be resolved. 

Promises will start doing whatever task you assign it as soon as the Promise constructor is invoked. This means Promises are _eager_.

A Promise can be in one of 3 possible states:
* Pending: not yet fulfilled or rejected

* Fulfilled: A result is available. `onFulfilled()` will be called (e.g., `resolve()` gets called)

* Rejected: An error occured. `onRejected()` will be called (e.g., `reject()` gets called)

If a Promise is not pending (it has been resolved or rejected), that Promise is __settled__. A settled Promise is immutable. Calling `resolve()` or `reject()` on a settled Promise will have no effect.

To implement a JavaScript Promise, the syntax looks like this:

```
new Promise( /* executor */ function(resolve, reject) { ... } );

/* or es6 */
new Promise((resolve, reject) => { ... } );
```

The _executor_ is a function that is passed with the arguments `resolve` and `reject`. The executor function is immediately executed by the Promise implementation, passing the `resolve` and `reject` functions. These functions, when called, resolve or reject the Promise, respectively.

All Promise instances get `then` and `catch` methods which allow you to react to the Promise.


Now, it's time for an extremely contrived example, using the infamous "Soup Nazi" experience from Seinfeld.

You want some amazing soup. In order to receive said soup, you have to follow some strict rules set by the person in charge of the soup.

1. You must immediately move to the right upon entering
2. Order your soup with no enthusiasm at all
3. Put your money on the counter and move left
4. Take your soup and do not make any comments

The soup maker __promises__ to fulfill your soup order, as long as those requirements are met.

I'll set up some variables to follow these rules.

```
let movingRight = true;
let enthusiastic = false;
let moneyOnCounter = true;
let movingLeft = true;
```

Then I'll create the first Promise. We are going to move to the right.

```
...

// promise
const moveToRight = new Promise(
  (resolve, reject) => {
    if (movingRight) {
      let soupToOrder = {
        name: 'Mulligatawny',
        price: '$3.99'
      };
      console.log("Moved to right.")
      resolve(soupToOrder); // promise is fulfilled
    } else {
      let reason = new Error('Did not move right. \nNo soup for you!')
      reject(reason); // promise is rejected
    }
  }
)
```

So what's going on here? We are defining a new Promise, `moveToRight`, and passing in our `resolve` and `reject` functions. _If_ we are moving to the right (`movingRight === true`), we have time to decide which soup we want to order, then we call `resolve()` and pass in our success value, `soupToOrder`. Our Promise is then fulfilled! If you can't follow the rules and don't go right, we'll call `reject()` and pass in our fail value. _No soup for you!_

Now that we have a Promise, lets consume it and get that soup!

```
...

// call our promise
const getSoup = () => {
  moveToRight
    .then(fulfilledSoup => { 
      // You followed the rules, here is your soup
      console.log(`Here is your ${fulfilledSoup.name}.`)
    })
    .catch(error => {
      // You broke the rules. No soup for you.
      console.log(error.message)
    })
};

getSoup();

/* output */
Here is your Mulligatawny.
```

1. We call our function, `getSoup`. In this function, we will consume our Promise `moveToRight`
2. We want to do something once the Promise is _resolved_ or _rejected_. We use `.then` and `.catch` to handle this.
3. In this example, we have `fulfilledSoup => {...}` in `.then`. The value of `fulfilledSoup` is the value you pass in your Promise `resolve(value)`. So in our case, that will be `soupToOrder`.
4. We have `error => {...}` in `.catch`. The `error` value is whatever you pass in your Promise `reject(value)`. It will be `reason` in this case.

## Chaining Promises

Promises can be chained.

So you've entered the restaurant and moved right. Now, with as little enthusiasm as possible, you order your soup.

We'll use slightly different syntax here, and call `Promise.resolve()`. The Promise API exposes several useful methods, `resolve()` being one of them. This creates a new Promise object that always resolves.

```
let enthusiastic = false; // ahem, curb your enthusiasm... I'll see myself out.
...

// 2nd promise
const orderSoup = soup => {
  if (!enthusiastic) {
    console.log(`${soup.price} for ${soup.name}.`)
    return Promise.resolve(soup) // fulfilled
  } else {
    let reason = new Error('Too enthusiastic. No soup for you!')
    return Promise.reject(reason) // rejected
  }
}
```

This time, we're passing in our soup object that gets resolved from our first Promise. If successful, it will return our Promise of soup. If an error occurs, it will return the defined reason.

Now let's chain the Promises. You can _only_ start the `orderSoup` process after the Promise to `moveToRight` has been fulfilled.

```
...

const getSoup = () => {
  moveToRight
    .then(orderSoup)
    .then(fulfilledSoup => {
      console.log(`Here is your ${fulfilledSoup.name}.`)
    })
    .catch(error => {
      console.log(error.message)
    })
}

getSoup();
```

We're calling `getSoup()`, which will consume our Promise to `moveToRight`. _Then_ we'll call `orderSoup` and pass it the result of the `resolve(soupToOrder)` from `moveToRight`. If the soup maker is satisfied, _then_ you receive your `fulfilledSoup`. We're just passing our soup object down the line.

If you obeyed the rules, you'll see:

```
Moved to right.
$3.99 for Mulligatawny.
Here is your Mulligatawny.
```

If you broke the rules, say you were enthusiastic, the `.catch` method kicks in and will process the error message:

```
let enthusiastic = true;
...

/* output */
Moved to right.
Too enthusiastic. No soup for you!
```

We really just want our soup! So we'll set `enthusiastic` back to `false` and get ready for our next Promise. We have to pay and move left.

```
...

// 3rd promise
const payAndMoveLeft = soup => {
  if (moneyOnCounter && movingLeft) {
    console.log("Put money on counter, moved left.")
    return Promise.resolve(soup)
  } else {
    let reason = new Error('Either did not put money on counter or move left. \nCould not follow rules. No soup for you!')
    return Promise.reject(reason)
  }
}
```

Now let's consume this 3rd Promise.

```
...

const getSoup = () => {
  moveToRight
    .then(orderSoup)
    .then(payAndMoveLeft)
    .then(fulfilledSoup => {
      console.log(`Here is your ${fulfilledSoup.name}.`)
    })
    .catch(error => {
      console.log(error.message)
    })
}

getSoup();
```

Again, we call `getSoup`, which calls `moveToRight`, which resolves an object (`soupToOrder`), _then_ passes the object to `orderSoup`, which then passes the object to `payAndMoveLeft`. Finally, you receive your `fulfilledSoup` object. _Whew!_

Assuming you followed the rules, you should see:

```
Moved to right.
$3.99 for Mulligatawny.
Put money on counter, moved left.
Here is your Mulligatawny.
```

At this point, we'll just pretend you accept your soup and give no comments, so you don't receive a one-year ban. If you're George, we could add a function to make your bread cost extra.

This is just a silly example to show the concepts behind Promises, and to demonstrate the _composability_ of Promises. They are a great way to avoid the dreaded 'callback hell.'

In the real world - calling APIs, downloading, and reading files are some common use cases of Promises.

I've been using a lot of Promises in AngularJS. Without getting too in depth on that subject, these promises look something like this:

```
function getData() {
  return $http.get('/route/to/get/data')
	          .then(doSomethingWithData)
}
```

AngularJS's `$http` API is based on the Deferred/Promise APIs exposed by the `$q` service. So in the above example - I'm calling `$http.get()` which generates an HTTP request and returns a Promise (the data we're looking for), _then_ handling that response data. 

That's all I've got for now. I have barely scratched the surface on this topic. I'm just exploring the power of Promises, and may make another post as I learn more about them. Happy coding!
