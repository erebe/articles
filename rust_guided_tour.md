# A Guided tour of Stream in Rust

When wanting to make GRPC or Websocket servers for Qovery infrastructure, I found a lot of guide regarding the inner working of futures, but not too many regarding how the Stream API is working in Rust, and more importantly how to use it properly.

Sadly, you can't turn a blind eye on Streams as when you go beyond the simple request/response protocol of our beloved REST Api, the notion of [flow](https://kotlinlang.org/docs/flow.html), [async generator](https://peps.python.org/pep-0525/) or more commonly [Stream](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API), arise naturally.

This is especially true for Rust, where when you decide to use tonic for your GRPC or tokio tungstenite
for your Websocket, the only usable interface with those librairies are around those Stream.

So this article is all about Stream and present them in the context of Rust



## What is a stream ? 

A stream is an iterator when seen from the [asynchronous world](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/). If you wander around the synchronous world, and you observe an iterator, it will look like something like

```
Iterator<MyItem>
```

Which represent a **sequence of 0..N objects of MyItem** that can be retrieved if asked nicely to the iterator.

Chances are, you already saw one in flesh. In [Java it exists since v1.2](https://docs.oracle.com/javase/8/docs/api/java/util/Iterator.html), in [C++](https://www.cplusplus.com/reference/iterator/) you have it in the standard librairy and also in [Python](https://wiki.python.org/moin/Iterator)

Iterator commonly appears when you want to iterate over a collection, like a list, vector, tree, … It is a **common abstraction that allow to decouple from the collection implementation** and represent the intent of a sequence of items, that can be retrieved in a linear fashion.

In Rust, an Iterator has only 2 requirements

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

An associated type Item, which represent the type of objects that our iterator is going to yield back, and a method next() that return an Option of the associated Item.

**Why an Option** ?
To inform you when receiving a None that the iterator does not have anymore element, and is now exhausted.
If you take the Java API for example, it has 2 methods. One called *next()* like in Rust but which return directly an item, and another one called *hasNext()*. 
It is up to the developer to call *hasNext()* before calling *next()*, and if you forgot to do that, it is a logical error that can cause your program to crash.
By merging *hasNext(*) and *next()*, and retuning an *Option()*, Rust prevents this kind of error and allows a safer api interface for the developer.



## Stream: An Asynchronous Iterator  

Now that we saw what an iterator is, let’s go back to our Stream, which we told is an asynchronous version of an iterator.
Ok, let’s check its definition

```rust
pub trait Stream {
    type Item;
    fn poll_next(
        self: Pin<&mut Self>,
        cx: &mut Context
    ) -> Poll<Option<Self::Item>>;
}
```

Hum, what are those poll/pin stuff ? Forget them for now, we can just rewrite the definition as this to better understand

```rust
pub trait Stream {
    type Item;
    fn next(self: &mut Self ) -> impl Future<Output = Option<Self::Item>>;
    // Warning, the trait does not really return a Future, it is more correct to say that Stream trait is also a Future
    // but for the sake of the explanation, we are going to say that next returns a future
}
```

Functions that return *Future* in Rust can be elided thanks to the special async keyword. 
So if we transform the definition again, it turns out the Stream definition is mentally equivalent to

```rust
pub trait Stream {
    type Item;
    async fn next(self: &mut Self) ->  Option<Self::Item>;
}
```

Here we can see that the Stream trait is equivalent to the Iterator trait, only with an async keyword in front of the function.

At first, the definition of Stream seems more complex that the Iterator trait, but this is only due to the inherent complexity/machinery necessary for async to work in Rust.

### Simplified explanation of Async in Rust

To allow you to understand what is going on in the original trait, we can briefly define those types:

**[Future](https://doc.rust-lang.org/std/future/trait.Future.html)** is a type that promise to yield some value in the future. It can be seen as a task, that is going to do some action in order to fulfil this promise and return the value


**[Poll](https://doc.rust-lang.org/std/task/enum.Poll.html)**, is the type that bridge the asynchronous world with the synchronous world. It can only have 2 variants

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

If Future can be seen as a task that return a value at some point, Poll is the return type of the future, indicating if the value is ready or not yet.
This type allow the runtime that is responsible for making the task/future progress (usually an executor, i.e: https://tokio.rs/) to known if the value is ready to be consumed or if the task/future should be re-polled at a later point

 
**[Contex](https://doc.rust-lang.org/std/task/struct.Context.html)** is a struct that only contain a Waker. To avoid the async runtime to poll your future again and again while it is not ready yet, which take cpu time and starve other future to be polled. The Waker type allows the future to notify back the runtime that it is ready to make some progress and so should be polled again
This is a common scenario where you don’t know when your value is becoming ready because it is waiting on an external factor. (i.e: a network socket with data ready to be read, getting the lock of a mutex), so instead of wasting cpu time, the future register the waker to be called by passing it to a reactor (kqueue, epoll, io completion) or something that get notified when some value is potentially ready.

For **[Pin](https://doc.rust-lang.org/std/pin/struct.Pin.html)**, just say briefly that it is a type that enforce your object to not move into memory. You can forget it when you see it, but if you want to dig deeper on this complex topic, I forward you to the article of [fasterthanli](https://fasterthanli.me/articles/pin-and-suffering) or the shorter [Cloudflare article](https://blog.cloudflare.com/pin-and-unpin-in-rust)

### Simplified flow of retrieving value from a future
![Simplified view of flow of a Future](/home/erebe/2022-05-03_14-03.png) 

In the schema above, we have a future that is reponsible to retrieve the value from a tcp socket. 
At first no bytes are ready, so it forward the waker to the reactor, which when he recieve some data call the Waker.wake() function which in turn notify the async runtime to poll again the task.


## How to create a Stream ?

Now that we have all the concept to grasp the overview of the theory, let’s go back to practicals examples for our Stream and see how to create some.

The easiest one is [Stream::iter](https://docs.rs/futures/latest/futures/stream/#functions), this function allows creating a stream from an iterator.
As a Stream is an async iterator, this function create a stream where every time it will be polled it returns a *Poll::Ready(next value from the iterator)*
Basically, the future is never yielding/awaiting, it is always ready when polled.

```rust
use futures::stream::{self, StreamExt};

let stream = stream::iter(vec![17, 19]);
assert_eq!(vec![17, 19], stream.collect::<Vec<i32>>().await);
```

*Stream::iter*, is a useful one for tests, where you don’t care about the async/await stuff but only interested in the stream of values.

Another interesting one is *repeat_with*, where you can pass a lambda/closure in order to generate the stream of values on demand/lazily

```rust
use futures::stream::{self, StreamExt};

// From the zeroth to the third power of two:
let mut curr = 1;
let mut pow2 = stream::repeat_with(|| { let tmp = curr; curr *= 2; tmp });

assert_eq!(Some(1), pow2.next().await);
assert_eq!(Some(2), pow2.next().await);
assert_eq!(Some(4), pow2.next().await);
assert_eq!(Some(8), pow2.next().await);
```

***
### Gotchas
If you took a look at the import, you will see that we use StreamExt, which is a shortcut for StreamExtension. It is a common practice in Rust to put only the minimal definition of a trait in a file and additional/helper/nicer api in another Extension file. In our case, all the goodies and easy to use functions live in the StreamExt module, and if you don’t import it, you will end-up only with poll_next from the Stream module, which is not really friendly.

Be aware also that Stream trait is not (yet) is the rust std::core like future. They live in the future_utils crate, and StreamExtension are not standard yet also. This often means that you can get confusing/conflicting import due to different library providing different one.
For example tokio provides different [StreamExt](https://docs.rs/tokio-stream/0.1.8/tokio_stream/trait.StreamExt.html than futures_utils)
If you can, try to stick to futures_utils, as it is the most commonly used crate for everything async/await

***


Ok, so far we only saw how to create stream from normal synchronous code, how do we manage to create stream that await something for real ?

A handy function for that is [Stream::unfold](https://docs.rs/futures/latest/futures/stream/fn.unfold.html). It takes into parameter a state/seed, and invoke an async future passing it that is responsible for generating the next element of the stream + the new state that is going to be passed.


For example, if we want to generate a stream that iterate over the paginated response of an api endpoint, we will do something like this one.

```rust
use futures::stream::{self, StreamExt};

let stream = stream::unfold(0, |page_nb| async move {
    if page_nb > 50 {
    return None;
   }
   
  let events = get_events_from_page(page_nb).await;
  Some((events, page_nb + 1))

});
```

You can even create a state machine or sequence of different action with unfold with

```rust
let state = (true, true, true);
let stream = stream::unfold(state, |state| async move {
   match state {
       (true, phase2, phase3) => {
            // do some stuff for phase 1
           let item = async { 1 }.await;
           Some((item, (false, phase2, phase3)))
       },
       (phase1, true, phase3) => {
           // do some stuff for phase 2
           let item = async { 2 }.await;
           Some((item, (false, false, phase3)))
       },
       (phase1, phase2, true) => {
            // do some stuff for phase 3
            let item = async { 3 }.await;
           Some((item, (false, false, false)))
       },
       _ => None,
   }
});

assert_eq!(Some(1), stream.next().await);
assert_eq!(Some(2), stream.next().await);
assert_eq!(Some(3), stream.next().await);
assert_eq!(None, stream.next().await);
```


Here, the state only old the phase we are in, and helps to unroll the seed/state in order to generate the correct computation for each step.

Manually unrolling our stream from a state is fun but a bit tedious if we have to do it by hand everytime. It would be nice to have something more high level like a generator in python where you have a special keyword `yield` that does all the magic for you

	# A generator in python with the special keyword yield
	def firstn(n):
	    num = 0
	    while num <= n:
	        yield num
	        num += 1

 
 
 And like for everything in Rust, if it is not in the language, there is a macro! for that.
 
 ## A higher level API
 
 The [async-stream](https://docs.rs/async-stream/0.3.3/async_stream/) crate provides 2 macros `stream!` and `try_stream!` that allows you to create stream like if it was normal rust code.
 

	let s = stream! {
	    for i in 0..3 {
	        yield i;
	     }
	};


if we re-take our example from before with 3 phases, it will be simply transformed like this

	let stream = stream! {
	   yield async { 1 }.await;
	   yield async { 2 }.await;
	   yield async { 3 }.await;
	});
	
	assert_eq!(Some(1), stream.next().await);
	assert_eq!(Some(2), stream.next().await);
	assert_eq!(Some(3), stream.next().await);
	assert_eq!(None, stream.next().await);
	
Much simpler to read right ?  [Internally](https://github.com/tokio-rs/async-stream/blob/6b2725f174716a29b5111f31846c0360433dae73/async-stream/src/async_stream.rs#L42) the macro is going to create a tiny channel on the thread local storage between the future you provide it and the Stream object. The macro replace all `yield` keyword by a send to this tiny channel.
Finally, when the stream is polled for the next item, it poll in turn the provided future and check after is there is something in the tiny channel for the stream to yield back.

It basically the same than with stream::unfold function, but with a bit more machinery in order to propose a more high level API where the responsability of propagating the state of stream is done for you.

***
### Gotchas

If you take a close look at the traits provided for [Stream](https://docs.rs/futures-util/latest/futures_util/stream/trait.Stream.html), you will see that there is not 1, not 2, but 3 traits that look like a Stream. Namely [Stream](https://docs.rs/futures-util/latest/futures_util/stream/trait.Stream.html) , [TryStream](https://docs.rs/futures-util/latest/futures_util/stream/trait.TryStream.html) and [FusedStream](https://docs.rs/futures-util/latest/futures_util/stream/trait.FusedStream.html). What are those ?

-  [Stream](https://docs.rs/futures-util/latest/futures_util/stream/trait.Stream.html), is the one you already know, but with one behavior change from its Iterator counter-part, that you should not re-poll the stream once it has returned a None indicating the Stream is exhausted. If you do so, you enter the realm of undefined behavior and the implementation is allowed to do what it is best. [See panics section for me details](https://docs.rs/futures-util/latest/futures_util/stream/trait.Stream.html#panics)

-  [FusedStream](https://docs.rs/futures-util/latest/futures_util/stream/trait.FusedStream.html) is the same thing than a Stream, but allows the user to know if the Stream is really exhausted after the first None or if it is safe to poll it again.  For example if you want to create a stream that is backed by a circular buffer. After the first iteration the FusedStream is going to return None, but it is safe again to re-poll the FusedStream after that in order to re-resume a new iterator of this buffer

- [TryStream](https://docs.rs/futures-util/latest/futures_util/stream/trait.TryStream.html) is a special trait tailored around Stream that produce Result<value, error>, they propose functions that allows to easily match and transform the inner result. You can see them as a more convenient API for stream that yield Result items


## How to consume a Stream ? 

The only function you will ever need is [next()](https://docs.rs/futures-util/latest/futures_util/stream/trait.StreamExt.html#method.next), which live in the StreamExt trait.

From that you are able to consume it in a regular for/loop fashion inside any async block. 
Below a simple async function that take a stream emitting int, and return the sum of it values.

```
use futures_util::{pin_mut, Stream, stream, StreamExt};

async fn sum(stream: impl Stream<Item=usize>) -> usize {

    pin_mut!(stream);
    let mut sum: usize = 0;
    while let Some(item) = stream.next().await {
        sum = sum + item;
    }

    sum
}
```

From that you can see how it is possible to send item into a channel, pass it to an other function that handle single elements, consume part of the stream, ...

***
### Gotchas
Don't forget to pin your stream before iterating on it, the compiler is going to tell you if you miss it, but will recommand to use *Box::pin*, which is not necessary if you can stack pin it. For that you can use the *pin_mut!* macro from futures_utils

***


Stream can also be consumed thanks to combinators, there are many of them, like the well known *map*, *filter*, *for_each*, *skip*, so I invite you to discover them directly from the [documentation](https://docs.rs/futures-util/latest/futures_util/stream/trait.StreamExt.html)

A useful one for debugging or simply logging, is the *[inspect](https://docs.rs/futures-util/latest/futures_util/stream/trait.StreamExt.html#method.inspect)* combinator. It allows you to pass a lambda that will take by ref each item emitted by the Stream, without consuming the item. 

```
let stream = stream::iter(vec![1, 2, 3]);
let mut stream = stream.inspect(|val| println!("{}", val));
    
assert_eq!(stream.next().await, Some(1));
// will print also in the console "1"
```

***
### Gotchas
If you use combinator from TryStream(Ext), be careful to read the documentation, especially for method that start by try_xxx. Some method are going to tells you  that the stream is exhausted at the first *Err()*, while some other not. The purpose of the TryStream trait is to allow to easily work around Result item, so all those method handle *Err()* somehow a bit like a snowflake with a special behavior for it. So avoid you some un-expected trouble, and read the documentation first !


***


Next article will be about websocket and stream ...
