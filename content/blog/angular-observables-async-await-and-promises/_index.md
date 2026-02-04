---
title: "Angular: Observables, async/await, and Promises, oh my!"
image: /blog/angular-observables-async-await-and-promises/cover.jpg
summary: "A practical guide to understanding RxJS Observables in Angular, comparing them to Promises, and using async/await for asynchronous operations."
first_published: 2018-01-09
---

# Angular: Observables, async/await, and Promises, oh my!

Coming from the pre-Angular2 Angular.js world, Angular (which is already at version 5 at the time of writing) can seem daunting with its insistence of using the  [Observer/Observable design pattern](https://en.wikipedia.org/wiki/Observer_pattern). Everywhere you look, things seem to return an  [RxJS Observable](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html)  instead of that nice familiar promise we all know (and maybe even love?). When trying to pick up Angular, this was super frustrating and my gut reaction was to use the very-handy `toPromise()` method that Observables provide and take the easy way out, but I convinced myself to learn it since I was sure there was a reason for all this madness. Now I think I finally understand it and I’d like to share what I found out.

----------

So let’s start off basic — why use observable at all and not rely entirely on promises? Well, that one’s easy — you can read  [this more detailed answer on stackoverflow](https://stackoverflow.com/questions/37364973/angular-promise-vs-observable)  but the gist of it is that Observables allow you to cancel an ongoing task, they allow you to return multiple things, and allow you to have multiple subscribers to a single Observable instance. It also simplifies having to retry.

So let’s say you had the following promise-based code:

```typescript
doAsyncPromiseThing()
  .then(() => console.log("I'm done!"))
  .catch(() => console.log("Error'd out"))
```

In “Observable land” this is really not any more complicated:

```typescript
doAsyncObservableThing()
  .subscribe(
     () => console.log("I'm done!"),
     () => console.log("Error'd out")
  )
```

## Your own first observable

A lot of guides I read dive into making HTTP requests right away with observables since that is pragmatically where you will use them most, but I want to do things a bit differently in this article and get at the meat of what Observables are like without trying to make it practical right away. So how do we simply return a value in an async way using Observable? Well there’s two ways — we can use the constructor of it, or use the create method, both of which do the same thing (they are aliases of each other).

By the way I’ll be basing the rest of this article around some of the helper methods here, so for this first example I’ll post the full component’s source so you can follow along easier in something like Plnkr if you want to.

```typescript
//our root app component
import {Component, NgModule} from '@angular/core'
import {BrowserModule} from '@angular/platform-browser'
import {Observable} from 'rxjs/Observable';

@Component({
  selector: 'my-app',
  template: `
    <div>
      <h2>Observable Example</h2>
      <ul>
        <li *ngFor="let message of messages">{{message}}</li>
      </ul>
    </div>
  `,
})
export class App {
  constructor() {
    this.initialTime = Date.now();
    this.messages = [];
    this.log = (m) => {
      const dateDifference = Date.now() - this.initialTime;
      this.messages.push(`${dateDifference}: ${m}`);
    };
    this.doAsyncObservableThing = new Observable(observer => {
      observer.next('Hello, observable world!');
      observer.complete();
    });

    this.doAsyncObservableThing.subscribe(
      this.log
    );
  }
}

@NgModule({
  imports: [ BrowserModule ],
  declarations: [ App ],
  bootstrap: [ App ]
})
export class AppModule {}
```

So when you run this you should see “Hello, observable world!” because the observable completes immediately. Note that my helper “log” method adds the number of microseconds since the app loaded to each message so we can get more interesting with the next example: things that take some time to finish. Before we get there though — take note of the fact that our observer has a next and complete method that we’re using in the example above. While it’s tempting to view “.subscribe()” as being akin to the “.then()” of a promise, it is far from the truth. The fact is that next() can be called multiple times as an observable can return multiple results. In fact, there are  **_infinite and finite observables_**. As the names imply, finite observables return a set number of results while infinite observables can go on forever.

I’d also like to add here that there’s very simple shorthands for many of these common tasks like what we just did above — for example instead of defining the observable as I had done there one can simply do:

```typescript
this.doAsyncObservableThing = Observable.of('Hello, observable world!');
```

Which does exactly the same thing as above, and is a lot simpler. This is akin to $q.when(‘Hello’) from the Angular.js world. Note that you need to add the following import to the top of your file to access some of these helpers in Angular:

```typescript
import 'rxjs/add/observable/of';
```

Another neat thing to note here is that  [you do not need to unsubscribe from finite observables](https://stackoverflow.com/questions/38008334/angular-rxjs-when-should-i-unsubscribe-from-subscription) — RxJS will take care of it for you. See what happens if you try the following:

```typescript
this.doAsyncObservableThing = new Observable(observer => {
  observer.next('Started');
  setTimeout(() => {
    observer.next('Hello, observable world!');
  }, 1000);
  setTimeout(() => {
    observer.next('Done');
    observer.complete();
  }, 2000);
});
```

What you’ll see is the following:

* 0: Started
* 1001: Hello, observable world!
* 2001: Done

Note that depending on the situation, you’re probably better off using something like  [delay()](https://www.learnrxjs.io/operators/utility/delay.html)  instead of setTimeout for timing purposes with your observables; I’m just using setTimeout here to show a point.

So because we are still using the subscribe method, we will stay subscribed and get our log function invoked every time as we expect, but we have no way of knowing when the observable fully completes even though this is a finite observable where we’re calling the “observer.complete()” method. So to rectify that, let’s modify our subscription slightly:

```typescript
this.doAsyncObservableThing = new Observable(observer => {
  observer.next('Started');
  setTimeout(() => {
    observer.next('Hello, observable world!');
  }, 1000);
  setTimeout(() => {
    observer.complete();
  }, 2000);
});

this.doAsyncObservableThing.forEach(
  this.log
).then(() => {
  this.log('Done');
});
```

In the above example I have changed the subscribe() method into a forEach(). The forEach() method returns… a promise! so we can simply do a .then() on the result of forEach() which will be invoked when the observable has fully completed. The rule of thumb is when you expect something to happen once and be done you should probably be using subscribe() and if you’re expecting multiple results you should probably be using forEach().

## Chaining Observables

One of the most annoying things for me to figure out when I started working with Observable was how to chain them together — specifically I had a scenario where I had two HTTP requests that needed to happen in sequence, with some processing of the results in-between.

The naive solution to this is to simply subscribe inside the subscription:

```typescript
this.doAsyncObservableThing = new Observable(observer => {
  setTimeout(() => {
    observer.next('Hello, observable world!');
    observer.complete();
  }, 1000);
});

this.doAsyncObservableThing.subscribe((val) => {
  this.log(val);
  this.doAsyncObservableThing.subscribe((val) => {
    this.log(val);
  });
});
```

Note that the above will work, you will see the following:

* 1001: Hello, observable world!
* 2001: Hello, observable world!

However the code is just plain ugly and anything even marginally more complicated will quickly grow to be unmaintainable. So what can we do? Well another naive solution is to just use the toPromise() method on the observables and chain them together that way, but again that’s taking the easy way out and not really thinking with Observables. So what to do? Well there’s a couple things you can do, but before we get to that it’s important to understand that  **unlike promises, you don’t keep chaining subscribes() similar to how you would chain then()’s**. Instead, there’s a few solutions depending on what you’re looking for.

## Start all tasks immediately, get the results one by one (merge)

First, let’s try importing merge using import ‘rxjs/add/operator/merge’;

Next let’s try

```typescript
this.doAsyncObservableThing('First')
  .merge(this.doAsyncObservableThing('Second'))
  .subscribe((v) => {
    this.log(v);
  });
```

Our result looks like this:

* 1002: First
* 1002: Second

The key takeaway from this experiment is that the callback in subscribe() is invoked twice, once for ‘First’ and once for ‘Second’ but the intervals are starting from the same time — the timing confirms both complete after one second.

## Observables in sequence, using async/await

Now, this next one is going to use toPromise() because I haven’t found a better way around this yet, but it uses so sparingly and still remains the cleanest way I have found to accomplish what we did with the nested subscribe()’s without actually nesting them and that is to use the async and await keywords. Firstly, we will have to move our code into the NgOnInit method because a constructor cannot be async and that’s where we’ve had our code so far; next let’s talk briefly about the async and await keywords.

Async is a keyword that denotes a method is allowed to use the await keyword, and that it returns a promise (you have to make sure to specify a return value that is Promise-compatible, e.g. Promise<YourResult> ). The await keyword suspends execution of the async function until the promise has resolved or rejected — it must always be followed by an expression that evaluates to a promise. So let’s see this in action:

```typescript
this.log(await this.doAsyncObservableThing('First').toPromise());
this.log(await this.doAsyncObservableThing('Second').toPromise());
```

Notice that the call to this.log() will be suspended until the expression involving await can be evaluated, which is only after the promise resolves. This means that our result looks like this:

* 1005: First
* 2009: Second

With the timings we expect, where the second observable doesn’t start until the first one finishes.

## Quirks

### route.queryParams doesn't work with await

When you read back what query parameters are in your current route, you must do so using an observable: route.queryParams; [here](https://angular.io/guide/router#query-parameters)  are the official docs for that. However, doing an await on the toPromise() of that Observable doesn’t work — your execution will stop at that point and it won’t continue further. It took me longer to figure this out than I would care to admit, but it turns out that queryParams is a case of an infinite Observable — subscribing to it will result in your subscription being invoked every time the query params change rather than getting the current query params at that moment only. This means that observer.complete() is never called by the internal mechanisms in it, which means an await operation on it will never complete. The trick here if you just want to get the query params once is to import either the take or first operator to get only the first result (the one at the time of execution) which looks like this:

```typescript
import 'rxjs/add/operator/take';
// ...
const queryParams = await this.route.queryParamMap.take(1).toPromise();
// access queryParams as needed
```

Thanks for reading, hope this article helped you! If you want to see the above code run live, check out  [this plnkr](http://embed.plnkr.co/FpBcaV/?show=preview,src%2Fapp.ts).

----------

## Further Reading

Here are some other topics to look at for learning Observable, and recommended further reading:

* Cancelling observables
* Unsubscribing from observables
* Handling errors and catching exceptions
* Writing unit tests that involve observables
* Using  [pipe() to apply map(), reduce(), and filter()](https://blog.hackages.io/rxjs-5-5-piping-all-the-things-9d469d1b3f44)  on observable results
* The concepts of  [“Cold” and “Hot” observables](https://blog.thoughtram.io/angular/2016/06/16/cold-vs-hot-observables.html)  (e.g. observables that only begin doing things once there are subscribers versus observables that do stuff right away, with or without subscribers)
* [distinctUntilChanged()](https://www.learnrxjs.io/operators/filtering/distinctuntilchanged.html)  and  [debounceTime()](https://www.learnrxjs.io/operators/filtering/debouncetime.html)
