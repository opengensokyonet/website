---
title: An async operation's resource and its waits
date: 2026-10-28
draft: false
authors:
  - name: yiyue@starspunworks.com
    link: mailto:yiyue@starspunworks.com
---

An async operation in Rust is a future, and a future holds two things for as
long as it exists. It holds its own progress: the compiler turns the body of
an async function into a state machine, so where an operation stands
between polls is kept without being written by hand, and an operation built
from others holds their progress as well. And it holds its way to the
resource it works on, a stream or a reference to one, from the moment it is
constructed, because a poll hands it nothing but a waker. Where it waits
follows from both: the innermost future that cannot proceed registers the
waker with the resource, and the code polling the outermost future never
needs to know why it stopped.

Decoders and encoders already take the resource out of the operation: a
decoder is handed the bytes at each call, by the code that drives it. They
also hand the waits back. When the bytes run out, a decoder returns, and its
caller has to know what to wait for. These are separate choices. Who keeps
the progress between steps, the compiler or the code itself, is not a third
one: it follows from the other two and from what the compiler can express,
and the post comes back to it.

We wrote [nitori](https://github.com/opengensokyonet/nitori), an
experimental library, while designing the networking stack of Open Gensokyo
Network, to try lending the resource while keeping the waits in the
operation: a future holds a handle instead of the resource, and the code
polling the outermost future fills it for the polls that need it and decides
how long the resource stays acquired. Operations are still written and
composed as async functions.

The post goes through the ways we know to make each choice, what each gives
up, and how they can be mixed within one operation. It does not rank them.

<!--more-->

## What a future holds

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
```

A future makes progress only while it is polled, and a poll is synchronous:
the future does what it can and returns `Ready` with its output or `Pending`.
The `Context` it receives carries a waker, which the future leaves wherever it
waits so that it is polled again, and on stable Rust nothing else.

The examples in this post use a QUIC stream. The networking stack we are
designing implements QUIC in user space, so the send and receive buffers of a
stream are memory, shared with the task that processes the connection's
packets. The two sides of a stream are separate values, as in quinn: a
`StreamReader` and a `StreamWriter`. Three accesses to them, as async
functions:

```rust
/// Reads into `output`, waiting until at least one byte has arrived; returns 0
/// at the end of the stream.
async fn read(stream: &mut StreamReader, output: &mut [u8]) -> io::Result<usize>;

/// Takes between one and `maximum` bytes out of the receive buffer, waiting
/// until at least one has arrived; returns `None` at the end of the stream.
async fn read_chunk(
    stream: &mut StreamReader,
    maximum: NonZeroUsize,
) -> io::Result<Option<Bytes>>;

/// Appends as much of `input` as there is room for, waiting until there is
/// room for at least one byte.
async fn write(stream: &mut StreamWriter, input: &[u8]) -> io::Result<usize>;
```

From them, writing a whole buffer, and writing a QUIC variable-length
integer, the encoding that HTTP/3 uses for the type and the length of a
frame:

```rust
/// Writes all of `input`.
async fn write_all(stream: &mut StreamWriter, mut input: &[u8]) -> io::Result<()> {
    while !input.is_empty() {
        let count = write(stream, input).await?;
        input = &input[count..];
    }
    Ok(())
}

/// Writes `value` in its shortest encoding: the top two bits of the first
/// byte give the length, 1, 2, 4 or 8 bytes, and the rest is the value in
/// network byte order.
async fn write_varint(stream: &mut StreamWriter, value: VarInt) -> io::Result<()> {
    let value = u64::from(value);
    let length: usize = match value {
        0..0x40 => 1,
        0x40..0x4000 => 2,
        0x4000..0x4000_0000 => 4,
        _ => 8,
    };
    let tag = u64::from(length.ilog2()) << (8 * length - 2);
    let encoding = (tag | value).to_be_bytes();
    write_all(stream, &encoding[8 - length..]).await
}
```

`VarInt` is an integer below 2^62, converted from and to the primitive integer
types.

Neither function says where it stands between polls, and both have to
remember it. When `write` waits for room, `write_all` is suspended in the
middle of its loop, and what is left of `input` is its state; `write_varint`
is suspended inside it, with its encoding next to it. The compiler stores
exactly this in the state machine it generates for each function, and the
future of `write_varint` contains the future of `write_all`, which contains
the future of `write`. Libraries often write such leaf operations by hand
instead: tokio's `write_all` returns a `WriteAll` struct that keeps the rest
of the buffer in a field and implements `Future` itself. Written as async
functions, as here, they need nothing of the kind, and neither do the
operations built on them: composing operations composes their state. The next poll goes down the same path, and each function
continues from the `await` where it stopped.

The future of `write_varint` also holds `stream`. It takes the reference when
it is constructed and keeps it until it completes or is dropped, through
every suspension, and its type borrows the stream for that long. It could not
be otherwise: a poll brings nothing but a waker, so whatever an operation
reaches its resource through has to be part of its own state. That holds
whatever form the resource takes:

| what the future holds                                                      | it can reach the resource    | the resource is acquired | when to acquire it is decided by |
| -------------------------------------------------------------------------- | ---------------------------- | ------------------------ | -------------------------------- |
| the resource, or an exclusive borrow of it, such as a socket its task owns | from construction to the end | throughout               | —                                |
| a lock guard                                                               | from the lock to the end     | from the lock to the end | the operation, where it locks    |
| a reference to shared state that each access locks, such as `StreamWriter` | from construction to the end | during each access       | each access                      |

`StreamWriter` and `StreamReader` are of the third kind. Like the streams of
quinn, whose `SendStream` locks the state of its connection each time it is
polled
([`send_stream.rs`](https://docs.rs/quinn/0.11.11/src/quinn/send_stream.rs.html)),
each holds a reference to state it shares with the packet-processing task,
and `write` locks that state, appends what fits, and unlocks.

The rows differ in how much of the future's life the resource is acquired.
They agree in the rest: what the future reaches the resource through is part
of the future for as long as it exists, and when the resource is acquired is
decided by code inside it.

The waits follow from the same arrangement. The access that finds no room
registers the waker with the stream and returns `Pending`, and every future
above it returns `Pending` in turn. The code polling the outermost future
needs to know neither what was awaited nor where; it polls again when it is
woken.

## A resource bound to the operation's life

Binding the resource to the future is usually right. It gets in the way in
two kinds of situation, and in both, the code that knows best when the
resource should be held is not the code that holds it. There is also a cost
that no arrangement of the binding removes, and the section ends with it.

### A shared resource

When the resource is shared, an operation must not keep it acquired while it
waits for something only another user of the resource can provide. A writer
that locked the stream once and kept the guard while it waited for room would
stop the packet-processing task, which makes room by sending, and each would
wait for the other. Shared resources are therefore reached in the third way,
acquired inside each access.

That has its own cost when operations are composed. An access knows where its
own poll begins and ends; it cannot know that it is one of many futures
polled in the same pass. Writing an HTTP/3 frame means writing its type and
its length with `write_varint` and then its payload with `write_all`. When
there is room, one poll of the task runs through the whole tree, and every
access locks the stream:

```text
poll of the task
└─ write_frame                       (or any operation built on write_varint)
   ├─ write_varint                   the frame type
   │  └─ write_all
   │     └─ write × n                lock · append · unlock, each
   ├─ write_varint                   the payload length
   │  └─ write_all
   │     └─ write × n                lock · append · unlock, each
   └─ write_all                      the payload
      └─ write × n                   lock · append · unlock, each
```

Each access is small, often a copy of a few bytes, and each takes a lock that
the packet-processing side contends for. When the accesses are this small,
acquiring the resource can cost as much as the access, and composing
operations multiplies the acquisitions. How much they add up to depends on
how finely the format cuts the work. A frame states its length, so its payload
goes in pieces as large as the room, and the accesses grow with the number of
frames rather than with the bytes inside them. QPACK's encoder stream, which
carries the instructions that build the decoder's table of header fields,
states no lengths: where an instruction ends is known only by parsing its
prefix integers and the lengths of its strings, so async code that reads it
reaches the resource for each integer and each string, and the acquisitions
are a visible share of the work.

Only the code polling the outermost future knows where a poll begins and
ends. Lending the resource to the whole composed operation for that poll
would let the accesses share one acquisition. That code is in the best
position to decide how long the resource stays acquired, and with the
resource bound inside the futures, it has no say in it.

### A resource that does not live as long

The binding also fixes how long the resource has to exist. A future that
holds `&'a mut StreamWriter` can exist only while that borrow does. This rules
out arrangements in which the resource is available one step at a time:

- The resource is handed to the code at each call, as a buffer passed as an
  argument, and not kept in between. This is how decoders work, and the next
  section is about them.
- The resource and the operations in progress on it are fields of the same
  value, such as a connection that owns the buffers of its streams. A future
  in one field that borrows another makes the value refer to itself.
- A task spawned on a runtime has to be `'static`, so it cannot borrow a
  resource owned elsewhere.

The usual way out of the last two is the third row of the table: the state
goes behind a lock, each operation holds a reference-counted handle to it,
and the acquisitions of the previous section come back.

### When the access itself is the cost

Sharing an acquisition helps only when the cost lies in acquiring the
resource. When every access is a system call, as on a kernel socket, each
one pays for entering the kernel and returning: the switch of privilege
level, the registers saved and restored, the mitigations against speculative
execution that run on the way, and the work every call repeats, such as
looking up the file descriptor, checking the arguments and copying the data
between user and kernel memory. However the operations are composed and
however long the code holds the socket, each access is still one system call,
and nothing in this post removes that cost.

io_uring reduces it with a structure close to the one this post is after. An
operation is written as an entry in a submission queue, in memory shared with
the kernel, instead of being performed; the code that drives the ring decides
when to enter the kernel, so the operations queued in between share one
entry, and files and buffers registered in advance spare each operation part
of the repeated work. Buffering in user space is the other answer, and it
brings back a resource that has to be acquired. Our stack takes that route: a
stream's buffers are memory behind a lock, and the system calls are made for
UDP datagrams, apart from the operations on streams.

## Waits handed to the caller

Decoders and encoders are the established way to keep the resource out of the
operation. A decoder is protocol code written as a state machine that is
handed bytes at each call and returns what it could decode; an encoder is
handed an item and a buffer to write it into. In Rust, the most common form
is the `Decoder` and `Encoder` traits of tokio-util's `codec` module:

```rust
fn decode(&mut self, src: &mut BytesMut) -> Result<Option<Self::Item>, Self::Error>;
fn encode(&mut self, item: Item, dst: &mut BytesMut) -> Result<(), Self::Error>;
```

### Bytes lent for one call

Their relation to the resource is the reverse of a future's. A decoder lives
as long as the protocol needs it, and sees the bytes only during a call of
`decode`, which borrows them for that call alone. The code that calls it
decides how long the resource stays acquired: a reader can lock the stream,
call `decode` on its buffer until it returns `None`, unlock, and then process
what it got.

The frames of the previous section, as a decoder that hands out the payloads
of frames of one type in pieces as they arrive:

```rust
/// Hands out the payloads of HTTP/3 frames of type `frame_type`, in pieces
/// as they arrive.
struct FrameDecoder {
    frame_type: VarInt,
    /// How much of the current payload is left, while inside one.
    remaining: Option<u64>,
}

impl FrameDecoder {
    fn new(frame_type: VarInt) -> Self {
        Self {
            frame_type,
            remaining: None,
        }
    }
}

impl Decoder for FrameDecoder {
    type Item = Bytes;
    type Error = io::Error;

    fn decode(&mut self, src: &mut BytesMut) -> io::Result<Option<Bytes>> {
        loop {
            match self.remaining {
                None => {
                    // Parsed from the start each time, until it is all there.
                    let Some((frame_type, length, header_length)) = parse_header(src)
                    else {
                        return Ok(None);
                    };
                    if frame_type != self.frame_type {
                        return Err(io::Error::new(
                            io::ErrorKind::InvalidData,
                            "unexpected frame type",
                        ));
                    }
                    src.advance(header_length);
                    self.remaining = Some(u64::from(length));
                }
                Some(0) => self.remaining = None,
                Some(_) if src.is_empty() => return Ok(None),
                Some(remaining) => {
                    let count = src
                        .len()
                        .min(usize::try_from(remaining).unwrap_or(usize::MAX));
                    self.remaining = Some(remaining - count as u64);
                    return Ok(Some(src.split_to(count).freeze()));
                }
            }
        }
    }

    fn decode_eof(&mut self, src: &mut BytesMut) -> io::Result<Option<Bytes>> {
        match self.decode(src)? {
            Some(piece) => Ok(Some(piece)),
            None if src.is_empty() && self.remaining.is_none() => Ok(None),
            None => Err(io::ErrorKind::UnexpectedEof.into()),
        }
    }
}
```

The two parsers work on a slice and return `None` when it does not hold a
whole header or integer; their bodies are left out:

```rust
/// Parses a frame's type and payload length, and returns them with the
/// length of the header, or `None` if the header is not all there.
fn parse_header(bytes: &[u8]) -> Option<(VarInt, VarInt, usize)>;

/// Parses the variable-length integer at the start of `bytes`, and returns
/// it with its length, or `None` if it is not all there.
fn parse_varint(bytes: &[u8]) -> Option<(VarInt, usize)>;
```

`FramedRead` is one way to drive it: it reads from an `AsyncRead` into a
`BytesMut` it owns, and calls `decode` on that, and `FramedWrite` does the
same for an encoder, writing out what the encoder put into its buffer. A
stream buffer in user space, like the one here, can be decoded where it is.
How sources and destinations of other shapes meet these traits, and what each
shape costs, is a topic for a later post.

### Progress kept in fields

How the work continues is the other difference from a future, and the code
above shows it. A future continues from the `await` where it stopped; `decode`
is called again from the top. Whatever the decoder needs in order to
continue, it keeps in its own fields: here `remaining`, which says that it is
inside a payload and how much of it is left, the job a local variable does in
an async loop. What it does not keep, it parses again: the header, until all
of it has arrived. Composition adds up the state. A decoder for a whole
request, built from a frame decoder and a decoder of field sections, holds
the state of both and its own record of where in the request it is; on every
call it passes bytes down, passes results up, and returns at whatever point
the bytes ran out. Large implementations, a quinn-proto connection among
them, show that this works at scale.

Writing the state by hand is how decoders are usually written, not something
their interface requires. The obstacle is that a state machine generated by
the compiler cannot easily borrow an input that is passed in afresh at each
step, which the next section is about, and [Mixing them](#mixing-them)
returns to a `Decoder` whose state the compiler keeps.

### The waits the interface names

What the interface does fix is where the waits are. This style is often
called sans-IO, and the
[definition](https://sans-io.readthedocs.io/how-to-sans-io.html) behind that
name excludes from the protocol code both network IO and "any form of
asynchronous flow control". The first is the resource; the second is the
waits, and handing them out has consequences of its own.

`decode` has no `Context`, so it cannot register a waker. When the bytes run
out, it returns `None`, and its caller has to know that this means "read more
from this source, then call again", and has to do the waiting itself. The
caller can act only on the waits it understands, so the interface names them,
and a `Decoder` names exactly one: more bytes from the source it decodes. It
has no way to wait for anything else. QPACK has such a case: a field section
on a request stream can refer to entries of the table whose instructions
have not yet arrived on the encoder stream, and decoding it has to wait for
that other stream
([RFC 9204](https://www.rfc-editor.org/rfc/rfc9204.html#name-blocked-streams)).
A `Decoder` on the request stream has no return value for that wait. It can
hand the section out undecoded or keep it aside, and its caller has to know
when to try again. Interfaces of larger implementations name more kinds of
waits: a quinn-proto connection reports the datagrams it has to send and the
time at which it has to be called back, and its caller provides both.

An `Encoder` names none. `encode` puts the whole item into `dst` in one call,
and returns nothing that could mean "not now". The waits of writing are still
there, and they move to whoever calls it. Waiting for room in the send buffer
becomes a buffer: `FramedWrite`, which connects an encoder to an
`AsyncWrite`, has the encoder write into a `BytesMut` that it grows as
needed, and writes it out as room appears. In `write_all`, what is left of the
input is the state of a partial write; here it is the encoder's output,
copied once more than `write_all` copies it. Waiting for flow control, which
decides how much of a stream may be sent at all, falls to the code that hands
over the items: it has to check before each one, and split a frame larger
than the window itself.

Either way, the caller does the waiting, and only for the waits the interface
names.

## Waits kept in the operation

The other direction lends the resource at each step, as a decoder is lent
its bytes, and keeps the waits where async code puts them, in the accesses:
the future holds a way to reach whatever resource is lent during the current
poll, and the code polling the outermost future lends it. An access that
touches the resource during a poll can still register a waker with it, so
what the operation waits for stays its own business, and composing operations
still composes their state.

The difficulty is getting a borrow into a state machine that the compiler
generates. A future receives nothing at each poll but the `Context`. A
coroutine receives a resume argument, but it cannot be generic over that
argument's lifetime
([rust-lang/rust#68923](https://github.com/rust-lang/rust/issues/68923)): a
borrow passed at one resume has to outlive the coroutine, so a coroutine
cannot be resumed with a fresh borrow each time, such as the `&mut BytesMut`
each call of `decode` is given. We know of four ways around this. Three are
described here in a few paragraphs each, as we either did not pursue them or
no longer do; the fourth, which nitori uses now, has the next section to
itself, and the rest of the post builds on it.

### A resource moved into a shared cell

On stable Rust, the future and the code that polls it can share a cell, such
as an `Rc<RefCell<..>>`. Before each poll, the caller moves the resource into
the cell; during the poll, the future reaches it through the cell; after the
poll, the caller moves it out again. This works only for what can be moved
out and back, such as the contents of a stream buffer, swapped out of the
lock guard for the length of a poll while the lock is held, and not for a
borrow, such as the guard itself. Every access is checked at run time, and
the type system does not say which futures expect the cell to be filled.

### A resource in the `Context`

The nightly `Context::ext` API
([rust-lang/rust#123392](https://github.com/rust-lang/rust/issues/123392))
lets the code that polls a future attach an extension to the `Context`, which
futures retrieve as `&mut dyn Any`. This is the most direct reading of "lend
the resource with each poll", and it changes no future signatures.

Its limitations follow from the type. `dyn Any` requires `'static`, so a view
that borrows the resource, such as a lock guard, cannot be attached without
erasing its lifetime unsafely, and every access is a downcast checked at run
time. The extension is visible to every future polled with that context,
whether or not it should touch the resource, and the type system does not
say which futures expect which extension. And a combinator that builds a new
`Context` for its children drops it: `FuturesUnordered`, for example, polls
each child with a context made by `Context::from_waker` around its own waker.

```text
poll(cx)                              cx carries the extension
└─ FuturesUnordered::poll
   └─ child.poll(child_cx)            child_cx = Context::from_waker(..):
                                      no extension
```

We did not implement this approach; this assessment is structural.

### Coroutines resumed with the resource

nitori's first implementation rewrote the body of each operation into a
coroutine with a procedural macro. Each time the code driving the operation
resumed it, it passed the scope of the current poll as the resume argument,
and awaiting a child operation passed the same argument on. The argument
carried no lifetime: it was a pointer to the scope with the lifetime erased,
valid only during the resume that passed it, and only code generated by the
macro used it. An access in the body went through that argument to the
scope, which acquired the resource at the first access of the poll and lent
it to every later one:

```text
poll of the task
└─ resume the operation with the scope of this poll
   └─ write_frame                      the bodies, rewritten into coroutines
      ├─ write_varint  access → scope: acquires the resource, lends it
      ├─ write_varint  access → scope: lends it
      └─ write_all     access → scope: lends it
   the poll ends, and the scope releases the resource
```

The macro had to see every suspension point, so the whole body was rewritten
and opaque macros inside it were rejected. Combinators such as `join!` and
`select!` could not drive children that needed the resource, because they
poll with a `Context` and have no way to pass the resume argument; awaiting
such a child went through the library's own protocol. Each capability needed
three extension traits written by hand: one to poll it on a resource, one to
start it as a child inside an operation, and one to run it on a resource.
Executors were coroutine operations too. It all required the `coroutines`
feature.

For a while the repository held this implementation next to an early version
of the handle implementation in the next section
([`prototypes` at commit
345b64f](https://github.com/opengensokyonet/nitori/tree/345b64fe5b4932b963241ebb48d0def3d3728f65/prototypes)).
The cost of an access was not what decided between the two. The current
implementation was chosen for the way operations are written and composed.

## A handle filled on demand

The current implementation keeps operations as ordinary async functions. An
operation does not hold the resource; it holds a handle, and a call (the
operation paired with a place for the resource) fills the handle with a view
of the resource for the polls that need one. The code polling the call
decides whether to lend a view, and which one.

`write_varint` as a call:

```rust
/// Writes `value` in its shortest encoding.
#[call]
async fn write_varint<F: Write, A: ReceiverAccess<Family = F>>(
    io: &mut A,
    value: VarInt,
) -> Result<(), WriteAllError<F::Error>> {
    let value = u64::from(value);
    let length: usize = match value {
        0..0x40 => 1,
        0x40..0x4000 => 2,
        0x4000..0x4000_0000 => 4,
        _ => 8,
    };
    let tag = u64::from(length.ilog2()) << (8 * length - 2);
    let encoding = (tag | value).to_be_bytes();
    io.write_all(&encoding[8 - length..]).await
}
```

The body is the one from the first section. `io` is the handle, and
`io.write_all` is nitori's own `write_all`, written like the one above
([`write.rs`](https://github.com/opengensokyonet/nitori/blob/d13516ddfae308adb3db6dcf1cf1a9ad65fcd6fd/crates/nitori_io/src/calls/write.rs)).
`F` is the family of the resource. A family names the type of the view lent
for one poll, `ReceiverView<'view>`, whose lifetime is that of the loan, and
capabilities such as `Write` are implemented on the family and take that
view. An operation therefore names neither the resource nor any of its
lifetimes, only the capabilities it needs (`Write` here), and whatever can
produce such a view runs it: a buffer the caller owns, an adapter over
`AsyncWrite`, or, through a lock executor, a value behind a mutex, whose
guard is composed into the value's own view. `F::Error` is the resource's
error.
The `#[call]` attribute keeps the function as written and generates the
methods that run it. Bodies use `.await`, `join!` and `select!` as usual.

Reading the header of a frame, with `read` from the same family and a
`read_exact` built on it:

```rust
/// Fills `buffer`. Returns `false` if the stream ends before the first byte,
/// and fails if it ends after it.
#[call]
async fn read_exact<F: Read, A: ReceiverAccess<Family = F>>(
    io: &mut A,
    mut buffer: &mut [u8],
) -> Result<bool, FrameError<F::Error>> {
    let length = buffer.len();
    while !buffer.is_empty() {
        // `read` fills a `BufMut`; a slice is one, and `read` moves it past
        // the bytes it fills.
        if io.read(&mut buffer).await? == 0 {
            return match buffer.len() == length {
                true => Ok(false),
                false => Err(FrameError::Truncated),
            };
        }
    }
    Ok(true)
}

/// Reads a variable-length integer, or returns `None` if the stream has
/// ended.
#[call]
async fn read_varint<F: Read, A: ReceiverAccess<Family = F>>(
    io: &mut A,
) -> Result<Option<VarInt>, FrameError<F::Error>> {
    let mut encoding = [0; 8];
    if !read_exact(io, &mut encoding[..1]).await? {
        return Ok(None);
    }
    let length = 1 << (encoding[0] >> 6);
    if !read_exact(io, &mut encoding[1..length]).await? {
        return Err(FrameError::Truncated);
    }
    let value = encoding[1..length]
        .iter()
        .fold(u64::from(encoding[0] & 0x3f), |value, &byte| {
            value << 8 | u64::from(byte)
        });
    Ok(Some(VarInt::try_from(value).expect("62 bits at most")))
}

/// Reads the header of a frame of type `frame_type` and returns the length of
/// its payload, or `None` if the stream has ended.
#[call]
async fn read_frame_header<F: Read, A: ReceiverAccess<Family = F>>(
    io: &mut A,
    frame_type: VarInt,
) -> Result<Option<u64>, FrameError<F::Error>> {
    let Some(actual) = read_varint(io).await? else {
        return Ok(None);
    };
    if actual != frame_type {
        return Err(FrameError::UnexpectedFrameType { actual });
    }
    let length = read_varint(io).await?.ok_or(FrameError::Truncated)?;
    Ok(Some(u64::from(length)))
}
```

`FrameError<E>` covers the failures of the accesses and of the frame itself;
its definition is left out.

Set against `FrameDecoder`, this shows what a call is: the decoder's relation
to its resource, with the control flow of the async functions of the first
section. The resource side is taken up below. On the control-flow side,
nothing is kept in fields. Where the operation stands is where it is suspended: when
the second integer of the header has not all arrived, `read_exact` is
suspended inside `read_varint` inside `read_frame_header`, and the next poll
continues there instead of parsing the type again. How much of a payload is
left is a local variable of the operation that reads it, in the next section,
as what was left of `input` was in `write_all`. And the operations
compose as async functions do: `read_frame_header` is `read_varint` twice,
and nothing in it says where a poll might end.

What the call holds between polls is its state and the handle, and the handle
holds no borrow of the resource: a view is lent for one poll and gone at its
end. The table of the first section gains a row:

| what the operation holds | it can reach the resource              | the resource is acquired                     | when to acquire it is decided by    |
| ------------------------ | -------------------------------------- | -------------------------------------------- | ----------------------------------- |
| a handle                 | during the polls it is lent a view for | during those polls, or until a release point | the code polling the outermost call |

This is the relation a decoder has to its bytes. A call's type can be named,
so it can be a field of a connection that lends another of its fields at each
poll; successive polls can lend views of different resources; and one view
can be lent to several calls within a pass, one after another. A call is
`Send` when everything its operation keeps apart from the handle is `Send`,
even if the resource is not.

The waits stay where they were. During a poll with a view, an access reaches
the resource itself, and when there is no room or nothing to read, it
registers the waker with the resource and returns `Pending`, as the accesses
of the first section do. Inside, the view reaches the operation through a
pointer whose lifetime the compiler does not check, as with the coroutines;
the handle is the safe interface around it.

For a resource behind a lock, the executor polls the call without a view. The
first access in a poll records a request and the poll returns; the executor
then acquires the lock and polls again with the view, which every access in
that poll shares. When an access has to wait, the poll ends and the executor
releases the view, so the packet-processing task can make room or deliver
bytes; the next poll acquires the view again. A poll that only awaits
something unrelated to the resource acquires nothing. nitori provides this
executor for `std::sync::Mutex`, `tokio::sync::Mutex` and
`futures::lock::Mutex`: a reference to the mutex runs an operation on the
value behind it, and the same `write_varint` runs on a buffer the caller
owns. The tree of the second section becomes:

```text
poll of the task
├─ poll the call without a view: the first write asks for one
├─ lock
├─ poll the call with the view
│  └─ write_frame
│     ├─ write_varint → write_all → write × n
│     ├─ write_varint → write_all → write × n
│     └─ write_all → write × n
└─ unlock
```

What is added: a request round trip on the first access of a poll that
starts without a view, and an executor that knows how to acquire the
resource. What is removed: the repeated acquisitions, and any acquisition
during polls that do not touch the resource.

## How long to hold it

Lending the view for a whole poll has its own cost. Everything the operation
does in that poll happens while the view is held, including computation that
does not need it: decoding a field section after reading it, or processing a
payload before reading the next frame. With a lock, other users wait for that
computation.

So an operation can mark a point after which it will not need the view for a
while. Reading a body's DATA frames and computing its CRC-32, for instance,
needs the stream only for the reads:

```rust
/// The type of HTTP/3 DATA frames.
const DATA_FRAME_TYPE: VarInt = VarInt::from_u32(0x00);

/// Reads DATA frames until the stream ends, and returns the pieces of the
/// body with its CRC-32.
#[call]
async fn read_body<
    F: ReadChunk<Chunk: AsRef<[u8]>>,
    A: ReceiverAccess<Family = F>,
>(
    io: &mut A,
) -> Result<(Vec<F::Chunk>, u32), FrameError<F::Error>> {
    let (mut body, mut crc) = (Vec::new(), crc32fast::Hasher::new());
    while let Some(mut remaining) = read_frame_header(io, DATA_FRAME_TYPE).await? {
        while let Some(maximum) =
            NonZeroUsize::new(usize::try_from(remaining).unwrap_or(usize::MAX))
        {
            let piece = io.read_chunk(maximum).await?.ok_or(FrameError::Truncated)?;
            remaining -= piece.remaining() as u64;
            io.release_view().await;
            crc.update(piece.as_ref());
            body.push(piece);
        }
    }
    Ok((body, crc.finalize()))
}
```

Each piece is taken while the view is held, and its CRC is computed after the
release point. The pieces are kept as they came, not copied into one buffer.
With two pieces ready in one poll:

```text
                       === view held
kept for the poll      piece 1 · CRC 1 · piece 2 · CRC 2
                       =================================
released at the point  piece 1 · CRC 1 · piece 2 · CRC 2
                       =======           =======
```

Whether releasing pays off depends on the resource, so both sides opt in.
The operation marks the release point; the executor decides at each point
whether to give the view back, for instance always, or only while someone
else is waiting for the lock. When it does, the operation continues at once
without the view and acquires a new one at its next access; the task is not
suspended. With a resource the task owns, a release point does nothing.

This puts the trade-off where the information is, but it does not make it
automatic: someone has to mark the release points, and an operation without
them holds the view for its whole poll.

## Mixing them

The two choices can be made differently at different levels of the same
code, and most code already does. Every decoder is driven by async code
somewhere: a reader that locks a stream and calls `decode` is an async
function that waits for bytes, and `FramedRead` is a `Stream` that does the
same. The decoder hands its waits to that code, which keeps them.

The boundary can also be crossed the other way. A decoder can hold a call and
lend it the bytes it is given at each call of `decode` as the view; when they
run out, the call suspends without registering a waker, and `decode` returns
`None`. We tried this with a `Decoder` that hands out payloads in pieces,
under `FramedRead`: it handed out the same pieces as the one written by hand,
and nothing it had parsed was parsed again. Its interface still hands the
waits to the caller; its progress is the compiler's. The same does not work
for an `Encoder`: a call inside `encode` could only write into the
destination it is given, which `FramedWrite` grows as needed, so it would
never wait.

Inside one async operation, the mix is finer. Part of the work is done
synchronously, as an encoder or a parser does it, and part through accesses,
and where that boundary lies is a decision separate from the two choices.
`write_varint` encodes the whole integer into an array on the stack,
synchronously, as an encoder would, and then writes the array with one
`write_all`. It could as well write each byte as it is produced:

```rust
/// Writes `value` in its shortest encoding, a byte at a time.
async fn write_varint_by_byte(
    stream: &mut StreamWriter,
    value: VarInt,
) -> io::Result<()> {
    let value = u64::from(value);
    let length: usize = match value {
        0..0x40 => 1,
        0x40..0x4000 => 2,
        0x4000..0x4000_0000 => 4,
        _ => 8,
    };
    let encoded = u64::from(length.ilog2()) << (8 * length - 2) | value;
    for index in (0..length).rev() {
        write_all(stream, &[(encoded >> (8 * index)) as u8]).await?;
    }
    Ok(())
}
```

Both are async operations, and both keep their waits; they differ in where
the synchronous work ends and the accesses begin. The second makes an access
for each byte and can stop between any two of them, with the loop index as
its state. The first makes one access for the whole integer, and the state of
a partial write is what is left of the array in `write_all`. On a stream that
locks in each access, that is up to eight acquisitions against one; written as
calls, both share the view of the poll, and what is left of the difference is
a future for each byte. Reading has the same choice: `read_varint` reads the
bytes and then decodes them with a fold over the array, a decoder in
miniature.

The boundary can be drawn at every level. A frame header could be encoded
whole into an array and written with one `write_all`, still without copying
the payload; a whole frame encoded into a buffer and then written is what
`FramedWrite` does, with the copy of the payload that comes with it. On the
reading side, an operation that holds the view could run a parser over the
bytes already buffered, as a decoder does, and await only when they end in an
incomplete unit. For a format like the QPACK encoder stream, that would
replace reading each integer through a future. It needs a capability that
lends the buffered bytes as a slice, which we have not tried.

## What decides the outcome

We measured these approaches on HTTP/2 and HTTP/3 workloads and on QPACK
encoder streams while developing nitori. Each way of writing the code differs
from the others in more than the two choices, the results turned on details
that a benchmark has to fix one way or another, and changing those details
changed which approach came out ahead. These are the details:

- What an access costs. A lock in user space costs little when nobody else
  holds it; a system call costs more.
- How finely the format cuts the work. With lengths, the accesses grow with
  the number of frames; without them, with each integer and each string.
- What happens while the resource is held. Decoding a field section or
  processing a body under the lock makes every other user of the stream wait.
  Releasing before processing, as at a release point, or splitting units off
  under the lock and processing them after, as a reader around a decoder
  does, keeps the hold short, and so does releasing after each piece instead
  of each frame.
- How the bytes are parsed. A parser over a contiguous slice reads an integer
  directly, while async code that reads it a byte at a time goes through a
  future for each byte. This is the boundary of the previous section,
  separate from both choices.
- The shape of the source. Whether its bytes are in one contiguous buffer, in
  separate pieces, or copied into memory the reader provides decides which
  approaches need a copy.
- The lock. A fair lock, an unfair one and a blocking one behave differently
  under contention. On a multi-threaded Tokio runtime, a pair of tasks
  contending for a `tokio::sync::Mutex` can fall into a pattern in which every
  unlock hands the lock directly to the other task and both end up running on
  one worker thread, and how much of a run is spent in that pattern varies
  from run to run. A blocking lock blocks a worker thread while it waits.
- Threads and contention. On one thread, a held lock delays only the tasks
  that run after it; with more threads, how long the resource stays acquired
  matters more than how often it is acquired.

One measurement does not depend on these details, because it isolates the
mechanism: one task on a current-thread runtime, a counter as the resource,
and a lock that is never contended
([`benches/call-bench` at commit
d13516d](https://github.com/opengensokyonet/nitori/tree/d13516ddfae308adb3db6dcf1cf1a9ad65fcd6fd/benches/call-bench)).
There, starting and finishing a call costs about 6 ns, and an uncontended
lock adds 12 to 22 ns to each access of ordinary async; a call that keeps the
view shares one acquisition among all the accesses of a poll.

## Limits of the handle approach

The handle approach has limits of its own:

- Defining operations requires nightly Rust, for `impl_trait_in_assoc_type`,
  and `yield_expr` for operations that yield items. Crates that only store,
  await or stream executions need no nightly features.
- The handle is borrowed mutably for each access, so one operation cannot
  `join!` two children that use the same resource at the same time. An
  operation on two resources takes a handle for each, and an executor over
  both, such as a pair of locks, acquires each for the polls that ask for it;
  two blocking locks then have to be asked for in one order.
- Release points are explicit, and one can be marked only where no child
  operation holds the handle; a child that yields items holds it until it
  finishes. An operation without release points holds the view for its whole
  poll.
- When an operation's arguments are iterators over borrowed data, the
  compiler fails to prove that its execution is `Send`, with a spurious
  higher-ranked lifetime error
  ([rust-lang/rust#100013](https://github.com/rust-lang/rust/issues/100013));
  having the operation own its data avoids the problem.

Questions we have not settled include whether release points can be placed
or chosen with less manual effort, how the fixed cost of a call can be
reduced further, and how operations that need two uses of the same resource
at once should be expressed.

## What each gives up

Each approach in this post, by how it reaches its resource, where its waits
are, who keeps its progress between steps, and what it needs:

| approach                                 | resource                                | waits                                                    | progress        | needs                                                                   |
| ---------------------------------------- | --------------------------------------- | -------------------------------------------------------- | --------------- | ----------------------------------------------------------------------- |
| a future holding the resource or a guard | held from construction or from the lock | in the operation                                         | the compiler    | the resource acquired while the operation waits                         |
| a future locking in each access          | a reference held from construction      | in the operation                                         | the compiler    | an acquisition for each access; a resource that outlives the future     |
| decoder or encoder                       | lent at each call                       | the caller's: one kind for `Decoder`, none for `Encoder` | usually by hand | the caller to wait; for `Encoder`, a buffer of its own and a copy       |
| shared cell                              | moved in for each poll                  | in the operation                                         | the compiler    | a resource that can be moved; checks at run time                        |
| `Context::ext`                           | attached to each poll's context         | in the operation                                         | the compiler    | nightly; a `'static` view; combinators that keep the context            |
| coroutines with a resume argument        | passed at each resume                   | in the operation                                         | the compiler    | nightly; an erased lifetime; the body rewritten; combinators of its own |
| call with a handle                       | lent for the polls that need it         | in the operation                                         | the compiler    | nightly; release points marked by hand                                  |

Where each one fits follows from the last two columns more than from speed:

- When a task owns its resource, or nothing else needs the resource while an
  operation waits, a future that holds the resource is the simplest. A call
  has nothing to save there.
- When the resource is shared and waiting must not hold it, a future that
  locks in each access works, and what it costs depends on how finely the
  format cuts the work.
- A decoder or an encoder fits when the waits are the ones its interface
  names, and its caller is in a position to do them. It runs on stable Rust,
  and a reader around it holds the resource only to split units off. Writing
  through `FramedWrite` costs a copy.
- A call fits when operations should be written and composed as async
  functions while their caller decides how long the resource stays acquired,
  or when the resource cannot outlive a step. The same operation runs on an
  owned buffer, an `AsyncWrite` adapter or a locked stream. Under contention,
  that decision has to include release points, or the call holds the
  resource through computation as well.
- The shared cell, `Context::ext` and the coroutines make the same two
  choices as a call and differ in the last column. The shared cell is the one
  of them that runs on stable Rust, for a resource that can be moved.

nitori is experimental and unpublished. The code is on
[GitHub](https://github.com/opengensokyonet/nitori). We welcome feedback on
the design in [its discussions](https://github.com/opengensokyonet/nitori/discussions).
