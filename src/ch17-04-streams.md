## Streams

So far in this chapter, we have mostly stuck to individual futures. The one big
exception was the async channel we used. Recall how we used the receiver for our
async channel in the [“Message Passing”][17-02-messages] earlier in the chapter.
The async `recv` method produces a sequence of items over time. This is an
instance of a much more general pattern, often called a *stream*.

A sequence of items is something we have seen before, when we looked at the
`Iterator` trait in Chapter 13, but there are two differences between iterators
and the async channel receiver. The first difference is the element of time:
iterators are synchronous, while the channel receiver is asynchronous. The
second difference is the API. When working directly with an `Iterator`, we call
its synchronous `next` method. With a `trpl::Receiver`, we call an asynchronous
`recv` method instead, but these APIs otherwise feel very similar.

That similarity is not a coincidence. A stream is like an asynchronous form of
iteration. Whereas the `trpl::Receiver` specifically waits to receive messages,
though, a general-purpose stream API needs to be much more general: it will just
provide the next item like `Iterator` does, but asynchronously. In fact, this is
roughly how it works in Rust, so we can actually create a stream from any
iterator. As with an iterator, we can work with a stream by calling its `next`
method, and then awaiting the output, as in Listing 17-30.

<Listing number="17-30" caption="Creating a stream from an iterator and printing its values" file-name="src/main.rs">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-async-await/listing-17-30/src/main.rs:stream}}
```

</Listing>

We start with an array of numbers, which we convert to an iterator and then call
`map` on to double all the values. Then we convert the iterator into a stream
using the `trpl::stream_from_iter` function. Then we loop over the items in the
stream as they arrive with the `while let` loop

Unfortunately, when we try to run the code, it does not compile. Instead, as we
can see in the output, it reports that there is no `next` method available.

<!-- TODO: fix up the path here? -->
<!-- manual-regeneration
cd listings/chapter-17-async-await/listing-17-30
cargo build
copy only the error output
-->

```console
error[E0599]: no method named `next` found for struct `Iter` in the current scope
 --> src/main.rs:8:40
  |
8 |         while let Some(value) = stream.next().await {
  |                                        ^^^^
  |
  = note: the full type name has been written to '/Users/chris/dev/rust-lang/book/listings/ch17-async-await/listing-17-30/target/debug/deps/async_await-bbd5bb8f6851cb5f.long-type-18426562901668632191.txt'
  = note: consider using `--verbose` to print the full type name to the console
  = help: items from traits can only be used if the trait is in scope
help: the following traits which provide `next` are implemented but not in scope; perhaps you want to import one of them
  |
1 + use futures_util::stream::stream::StreamExt;
  |
1 + use std::iter::Iterator;
  |
1 + use std::str::pattern::Searcher;
  |
1 + use trpl::StreamExt;
  |
help: there is a method `try_next` with a similar name
  |
8 |         while let Some(value) = stream.try_next().await {
  |                                        ~~~~~~~~

For more information about this error, try `rustc --explain E0599`.
```

As the output suggests, the problem is that we need the right trait in scope to
be able to use the `next` method. Given our discussion so far, you might
reasonably expect that to be `Stream`, but the trait we need *here* is actually
`StreamExt`. The `Ext` there is for “extension”: this is a common pattern in
the Rust community for extending one trait with another.

You might be wondering why `StreamExt` instead of `Stream`, and for that matter
whether there is a `Stream` type at all. Briefly, the answer is that throughout
the Rust ecosystem, the `Stream` trait defines a low-level interface which
effectively combines the `Iterator` and `Future` traits. The `StreamExt` trait
supplies a higher-level set of APIs on top of `Stream`, including the `next`
method and also many other utility methods like those from `Iterator`. We will
return to the `Stream` and `StreamExt` traits in a bit more detail at the end of
the chapter. For now, this is enough to let us keep moving.

All we need to do here is add a `use` statement for `trpl::StreamExt`, as in
Listing 17-31.

<Listing number="17-31" caption="Successfully using an iterator as the basis for a stream" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-31/src/main.rs}}
```

</Listing>

With all those pieces put together, things work the way we want! What is more,
now that we have `StreamExt` in scope, we can use all of its utility methods,
just like with iterators. For example, in Listing 17-32, we use the `filter`
method to filter out everything but multiples of three and five.

<Listing number="17-32" caption="Filtering a `Stream` with the `StreamExt::filter` method" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-32/src/main.rs}}
```

</Listing>

Of course, this is not very interesting. We could do that with normal iterators
and without any async at all. So let’s look at some of the other things we can
do which are unique to streams.

### Composing Streams

Lots of things are naturally represented as streams: items becoming available in
a queue, or working with more data than can fit in a computer’s memory by only
pulling chunks of it from the file system at a time, or data arriving over the
network over time. Because streams are futures, we can use them with any other
kind of future, too, and we can combine them in interesting ways. For example,
we can batch up events to avoid triggering too many network calls, set timeouts
on sequences of long-running operations, or throttle user interface events to
avoid doing needless work.

Let’s start by building a little stream of messages, similar to what we might
see from a WebSocket or other real-time communication protocols. In Listing
17-33, we  create a function `get_messages` which returns `impl Stream<Item =
String>`. For its implementation, we create an async channel, loop over the
first ten letters of the English alphabet, and send them across the channel.

We also use a new type: `ReceiverStream`, which converts the `rx` receiver from
the `trpl::channel` into a `Stream` with a `next` method. Back in `main`, we use
a `while let` loop to print all the messages from the stream.

<Listing number="17-33" caption="Using the `rx` receiver as a `ReceiverStream`" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-33/src/main.rs:all}}
```

</Listing>

When we run this code, we get exactly the results we would expect:

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
Message: 'a'
Message: 'b'
Message: 'c'
Message: 'd'
Message: 'e'
Message: 'f'
Message: 'g'
Message: 'h'
Message: 'i'
Message: 'j'
```

We could do this with the regular `Receiver` API, or even the regular `Iterator`
API, though. Let’s add something that requires streams, like adding a timeout
which applies to every item in the stream, and a delay on the items we emit.

In Listing 17-34, we start by adding a timeout to the stream with the `timeout`
method, which comes from the `StreamExt` trait. Then we update the body of the
`while let` loop, because the stream now returns a `Result`. The `Ok` variant
indicates a message arrived in time; the `Err` variant indicates that the
timeout elapsed before any message arrived. We `match` on that result and either
print the message when we receive it successfully, or print a notice about the
timeout. Finally, notice that we pin the messages after applying the timeout to
them, because the timeout helper produces a future which needs to be pinned to
be polled.

<Listing number="17-34" caption="Using the `StreamExt::timeout` method to set a time limit on the items in a stream" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-34/src/main.rs:timeout}}
```

</Listing>

However, since there are no delays between messages, this timeout does not
change the behavior of the program. Let’s add a variable delay to the messages
we send. In `get_messages`, we use the `enumerate` iterator method with the
`messages` array so that we can get the index of each item we are sending along
with the item itself. Then we apply a 100 millisecond delay to even-index items
and a 300 millisecond delay to odd-index items, to simulate the different delays
we might see from a stream of messages in the real world. Because our timeout is
for 200 milliseconds, this should affect half of the messages.

<Listing number="17-35" caption="Sending messages through `tx` with an async delay without making `get_messages` an async function" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-35/src/main.rs:messages}}
```

</Listing>

To sleep between messages in the `get_messages` function without blocking, we
need to use async. However, we cannot make `get_messages` itself into an async
function, because then we would return a `Future<Output = Stream<Item =
String>>` instead of just a `Stream<Item = String>>`. The caller would have to
await `get_messages` itself to get access to the stream. But remember:
everything in a given future happens linearly; concurrency happens *between*
futures. Awaiting `get_messages` would require it to send all the messages,
including sleeping between sending each message, before returning the receiver
stream. As a result, the timeout would end up useless. There would be no delays
in the stream itself: the delays would all happen before the stream was even
available.

Instead, we leave `get_messages` as a regular function which returns a stream,
and spawn a task to handle the async `sleep` calls.

> Note: calling `spawn_task` like this works because we already set up our
> runtime. Calling this particular implementation of `spawn_task` *without*
> first setting up a runtime will cause a panic. Other implementations choose
> different tradeoffs: they might spawn a new runtime and so avoid the panic but
> end up with a bit of extra overhead, or simply not provide a standalone way to
> spawn tasks without reference to a runtime. You should make sure you know what
> tradeoff your runtime has chosen and write your code accordingly!

Now our code has a much more interesting result! Between every other pair of
messages, we see an error reported: `Problem: Elapsed(())`.

<!-- manual-regeneration
cd listings/listing-17-35
cargo run
copy only the program output, *not* the compiler output
-->

```text
Message: 'a'
Problem: Elapsed(())
Message: 'b'
Message: 'c'
Problem: Elapsed(())
Message: 'd'
Message: 'e'
Problem: Elapsed(())
Message: 'f'
Message: 'g'
Problem: Elapsed(())
Message: 'h'
Message: 'i'
Problem: Elapsed(())
Message: 'j'
```

The timeout does not prevent the messages from arriving in the end—we still get
all of the original messages. This is because our channel is unbounded: it can
hold as many messages as we can fit in memory. If the message does not arrive
before the timeout, our stream handler will account for that, but when it polls
the stream again, the message may now have arrived.

You can get different behavior if needed by using other kinds of channels, or
other kinds of streams more generally. Let’s see one of those in practice in our
final example for this section, by combining a stream of time intervals with
this stream of messages.

### Merging Streams

First, let’s create another stream, which will emit an item every millisecond if
we let it run directly. For simplicity, we can use the `sleep` function to send
a message on a delay, and combine it with the same approach of creating a stream
from a channel we used in `get_messages`. The difference is that this time, we
are going to send back the count of intervals which has elapsed, so the return
type will be `impl Stream<Item = u32>`, and we can call the function
`get_intervals`.

In Listing 17-36, we start by defining a `count` in the task. (We could define
it outside the task, too, but it is clearer to limit the scope of any given
variable.) Then we create a an infinite loop. Each iteration of the loop
asynchronously sleeps for one millisecond, increments the count, and then sends
it over the channel. Since this is all wrapped in the task created by
`spawn_task`, all of it will get cleaned up along with the runtime, including
the infinite loop.

<Listing number="17-36" caption="Creating a stream with a counter that will be emitted once every millisecond" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-36/src/main.rs:intervals}}
```

</Listing>

This kind of infinite loop, which only ends when the whole runtime gets torn
down, is fairly common in async Rust: many programs need to keep running
indefinitely. With async, this does not block anything else, as long as there is
at least one await point in each iteration through the loop.

Back in our main function’s async block, we start by calling `get_intervals`.
Then we merge the `messages` and `intervals` streams with the `merge` method.
Finally, we loop over that combined stream instead of over `messages` (Listing
17-37).

<Listing number="17-37" caption="Attempting to merge streams of messages and intervals" file-name="src/main.rs">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-async-await/listing-17-37/src/main.rs:main}}
```

</Listing>

At this point, neither `messages` nor `intervals` needs to be pinned or mutable,
because both will be combined into the single `merged` stream. However, this
call to `merge` does not compile! (Neither does the `next` call in the `while
let` loop, but we will come back to that after fixing this.) The two streams
have different types. The `messages` stream has the type `Timeout<impl
Stream<Item = String>>`, where `Timeout` is the type which implements `Stream`
for a `timeout` call. Meanwhile, the `intervals` stream has the type `impl
Stream<Item = u32>`. To merge these two streams, we need to transform one of
them to match the other.

In Listing 17-38, we rework with the `intervals` stream, since `messages` is
already in the basic format we want and has to handle timeout errors. First, we
can use the `map` helper method to transform the `intervals` into a string.
Second, we need to match the `Timeout` from `messages`. Since we do not actually
*want* a timeout for `intervals`, though, we can just create a timeout which is
longer than the other durations we are using. Here, we create a 10-second
timeout with `Duration::from_secs(10)`. Finally, we need to make `stream`
mutable, so that the `while let` loop’s `next` calls can iterate through the
stream, and pin it so that it is safe to do so.

<!-- We cannot directly test this one, because it never stops. -->

<Listing number="17-38" caption="Aligning the types of the the `intervals` stream with the type of the `messages` stream" file-name="src/main.rs">

```rust,ignore
{{#rustdoc_include ../listings/ch17-async-await/listing-17-38/src/main.rs:main}}
```

</Listing>


That gets us *almost* to where we need to be. Everything type checks. If you run
this, though, there will be two problems. First, it will never stop! You will
need to stop it with <span class="keystroke">ctrl-c</span>. Second,  the
messages from the English alphabet will be buried in the midst of all the
interval counter messages:

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the tasks running differently rather than
changes in the compiler -->

```text
--snip--
Interval: 38
Interval: 39
Interval: 40
Message: 'a'
Interval: 41
Interval: 42
Interval: 43
--snip--
```

Listing 17-39 shows one way to solve these last two problems. First, we use the
`throttle` method on the `intervals` stream, so that it does not overwhelm the
`messages` stream. Throttling is a way of limiting the rate at which a function
will be called—or, in this case, how often the stream will be polled. Once every
hundred milliseconds should do, since that is in the same ballpark as how often
our messages arrive.

To limit the number of items we will accept from a stream, we can use the `take`
method. We apply it to the *merged* stream, because we want to limit the final
output, not just one stream or the other.

<Listing number="17-39" caption="Using `throttle` and `take` to manage the merged streams" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-39/src/main.rs:throttle}}
```

</Listing>

Now when we run the program, it stop after pulling twenty items from the stream,
and the intervals do not overwhelm the messages. We also do not get `Interval:
100` or `Interval: 200` or so on, but instead simply get `Interval: 1`,
`Interval: 2`, and so on—even though we have a source stream which *can* produce
an event every millisecond. That is because the `throttle` call produces a new
stream, wrapping the original stream, so that the original stream only gets
polled at the throttle rate, not its own “native” rate.  We do not have a bunch
of unhandled interval messages we are simply choosing to ignore. Instead, we
never produce those interval messages in the first place! This is the inherent
“laziness” of Rust’s futures at work again, allowing us to choose our
performance characteristics.

<!-- manual-regeneration
cd listings/listing-17-39
cargo run
copy and paste only the program output
-->

```text
Interval #1
Message: 'a'
Interval #2
Interval #3
Problem: Elapsed(())
Interval #4
Message: 'b'
Interval #5
Message: 'c'
Interval #6
Interval #7
Problem: Elapsed(())
Interval #8
Message: 'd'
Interval #9
Message: 'e'
Interval #10
Interval #11
Problem: Elapsed(())
Interval #12
```

There is one last thing we need to handle: errors! With both of these
channel-based streams, the `send` calls could fail when the other side of the
channel closes—and that is just a matter of how the runtime executes the futures
which make up the stream. Up till now we have ignored this by calling `unwrap`,
but in a well-behaved app, we should explicitly handle the error, at minimum by
ending the loop so we do not try to send any more messages!  Listing 17-40 shows
a simple error strategy: print the issue and then `break` from the loops. As
usual, the correct way to handle a message send error will vary—just make sure
you have a strategy.

<Listing number="17-40" caption="Handling errors and shutting down the loops">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-40/src/main.rs:errors}}
```

</Listing>

That is a good note to turn to our final section and wrap up this walk through
async in Rust, by discussing how futures (including streams), tasks, and threads
relate to each other, and how you can use them together.

{{#quiz ../quizzes/async-04-streams.toml}}


[17-02-messages]: ch17-02-concurrency-with-async.html#message-passing
