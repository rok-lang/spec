# 4.13. Streams (`async yield`)

Using the two modifiers `async` and `yield` together defines an asynchronous generator, a function that emits multiple values over time. Its result is a value of the library type `AsyncStream<T>` (implementing `Stream<Item = T>`), or `TryAsyncStream<T, E>` with `throws (E)` ([4.1.11](401-function-declaration.md#result-types-of-coroutine-functions)).

## 4.13.1. Three definition forms

The grammar for defining streams has three forms.

```
stream_fn       ::= "async" "yield" fn_decl
stream_block    ::= "async" "yield" block
stream_closure  ::= "async" "yield" closure_expr
```

```rok
async yield fn name(args) -> R { ... await ...; ... yield ...; }   // named
async yield { ... await ...; yield ...; }                           // inline block
async yield || { ... await ...; yield ...; }                        // closure modifier
```

These are the three forms of [generators](411-generators.md) and [async](412-async.md) working together.

## 4.13.2. Relationship of sync, async, generators and streams

The four forms relate as in the following table.

| Form             | Call result                         | Meaning                    |
|------------------|-------------------------------------|----------------------------|
| `fn`             | `R`                                 | one value immediately      |
| `async fn`       | `AsyncFuture<R>` | one value in the future    |
| `yield fn`       | `GenIter<R>` | multiple values synchronously |
| `async yield fn` | `AsyncStream<R>`   | multiple values over time  |

These are all combinations of two orthogonal dimensions: deferral (sync/async) and number of values (single/multiple).

## 4.13.3. Two keywords in the body

Both `await` and `yield` may be used in the body of an `async yield` function.

```rok
async yield fn read_lines(file: str) -> str {
    let f = await open(file);
    loop {
        let line = await f.read_line();
        match line {
            l: str => yield l,
            none   => break,
        }
    }
}
```

`await expr;` suspends the function until the Future completes; `yield expr;` produces a value and suspends until the Stream advances to the next step. The two suspension mechanisms coexist in one function, and the coroutine frame tracks which kind of suspension happened at which point.

## 4.13.4. Calling and consuming

Calling and consuming a stream looks like this.

```rok
async fn run() {
    let s = read_lines("data.txt");   // s: `AsyncStream<str>`

    for await line in s {              // asynchronous for
        process(line);
    }
}
```

The call itself immediately returns a **Stream** object; actual consumption happens in an asynchronous `for` loop, the `for await` form.

`for await` is the asynchronous variant of the `for` of [loops](../003-expressions/313-loop-expressions.md). Each iteration includes an `await` and waits asynchronously for the next value from the stream.

## 4.13.5. The `for await` form

The basic form of `for await` is:

```rok
for await pat in stream { body }
```

`for await` is a keyword combination for asynchronous iteration. Each iteration has the effect of `await stream.next()`, and the body is written like an ordinary `for` body.

This form may be used only inside the bodies of `async fn`, `async {}` and `async ||`.

## 4.13.6. Evaluation flow of `yield` and `await`

The following example shows the two keywords working together.

```rok
async yield fn paginate() -> Page {
    let next = "/start";
    loop {
        let page = await fetch(next);
        yield page;
        match page.next_url {
            url: str => next = url,
            none     => break,
        }
    }
}
```

First, `await fetch(...)` waits until the page is received; then `yield page` produces that page and suspends until the consumer requests the next value. When the consumer requests the next value, the function resumes and fetches the next page.

## 4.13.7. The Stream trait

The result object is the standard library's `AsyncStream<T>` (implementing `Stream<Item = T>`). Semantically it combines Iterator and Future: a mechanism that waits for the next value.

The detailed signatures and methods (`map`, `filter`, `take`, `merge`, `zip`, etc.) are defined in the standard library.

## 4.13.8. Combining with library functions

Stream transformation and combination are provided by standard library functions.

```rok
stream::merge(s1, s2)             // merges two streams
stream::zip(s1, s2)               // pairs up and advances together
stream::throttle(s, delay)        // rate limiting
```

The detailed functions and methods are defined in the standard library specification.

## 4.13.9. Infinite streams

A stream may also continue indefinitely.

```rok
async yield fn ticks() -> Time {
    loop {
        yield now();
        await sleep(1.second());
    }
}

async fn run() {
    for await t in ticks() {
        if should_stop() { break; }
        print(t);
    }
}
```

Such an infinite stream is terminated when the consumer exits with `break` or through a library transformation such as `take(n)`.

## 4.13.10. Comparison with channels and pipes

The standard library's `Channel<T>` is also a mechanism that moves multiple values over time, but it differs from streams as follows.

| Aspect                  | `Stream<Item = T>`              | `Channel<T>`                       |
|-------------------------|---------------------------------|------------------------------------|
| Definition              | result of an `async yield` function | library type                   |
| Producer and consumer   | integrated in one function      | two separate endpoints             |
| Many-to-many            | separate combination (merge, etc.) | natural (multiple senders)      |

In short, `Stream` models one asynchronous computation producing values over time, while `Channel` models multiple concurrent tasks exchanging values. The two abstractions complement each other.

## 4.13.11. `async` `yield` on methods

`async yield` may also be defined freely as a method.

```rok
impl Database {
    async yield fn iter_rows(self, sql: str) -> Row {
        let cursor = await self.query(sql);
        loop {
            let row = await cursor.next();
            match row {
                r: Row => yield r,
                none   => break,
            }
        }
    }
}

async fn run(db: Db) {
    for await row in db.iter_rows("SELECT ...") {
        process(row);
    }
}
```

Like ordinary methods, `async yield` methods may be defined and used freely.
