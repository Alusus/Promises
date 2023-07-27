# Promises
[[عربي]](readme.ar.md)

A simple Promise library for Alusus Language.

This library provides basic Promises functionality that is independent of the platform where the promises are to be
used (web, backend, etc). It does not provide an event loop; instead, the platform should provide an event loop
with support for this library.

## Adding to the Project

We can install this library using the following statements:

```
import "Apm";
Apm.importFile("Alusus/Promises");
```

## Example

```
import "Srl/Console";
import "Srl/errors";
import "Apm";
Apm.importFile("Alusus/Promises");

use Srl;
use Promises;

// define a class that represents a number, for using it in the next examples 
class Num {
    def val: Int;
    handler this~init() this.val = 0;
    handler this~init(i: Int) this.val = i;
    handler this~init(i: ref[Num]) this.val = i.val;
    handler this = ref[Num] this.val = value.val;
}

func testPromise {
    Console.print("Test Promise:\n");
    // create a promise with `Num` as the return type
    def promise: SrdRef[Promise[Num]] = Promise[Num].new();
    Console.print("promise 1 - status: %d, value: %d\n", promise.status, promise.result.val);

    // resolve the promise with the result as an object from class Num with the value 5
    promise.resolve(Num(5));
    Console.print("promise 1 - status: %d, value: %d\n", promise.status, promise.result.val);

    // reject the promise with an error that has the code 1 and an appropriate message
    promise.reject(castSrdRef[SrdRef[GenericError]().{
        construct();
        code = "example_err1";
        message = String("Unknown error 1");
    }, Error]);
    Console.print("promise 1 - status: %d, error: %ld\n", promise.status, promise.error.obj~ptr);
}
testPromise();

func testPromiseThen {
    Console.print("\nTest Promise.then:\n");
    // create a promise with `Num` as the return value
    def promise: SrdRef[Promise[Num]] = Promise[Num].new();

    // determine what needs to be executed after executing the promise
    // here we will print a message and resolve the promise
    def then: SrdRef[Promise[String]] = promise.then[String](
        closure (input: Num, p: ref[Promise[String]]) {
            Console.print("ThenPromise triggered\n");
            p.resolve(String("ThenPromise - received value: ") + input.val);
        }
    );
    Console.print("status: %d, value: %s\n", then.status, then.result.buf);
    promise.resolve(Num(6));
    Console.print("status: %d, value: %s\n", then.status, then.result.buf);
}
testPromiseThen();
```

## Functions and Types

### Promise

```
class Promise [ResultType: type] {
    def status: Int = Status.NEW;
    def result: ResultType;
    def error: SrdRef[Error];
}
```
A template Promise class where the promise result type is specified by the template argument.

`status` the current state of the promise.

`result` the result returned from the promise.

`error` the error that happened while executing the promise.

### resolve

```
handler this.resolve(res: ResultType);
```

Resolve the promise by switching its status from `NEW` to `RESOLVED` and storing the result.

parameters:

`res` the result to be stored in the promise as the execution result.

```
handler this.resolve(p: SrdRef[Promise[ResultType]]);
```

Resolve the promise using another promise. The current promise will wait for the given
promise to be complete and will carry its result.

`p` the promise whose status will eventually propagate to the current promise.

### reject

```
handler this.reject(err: SrdRef[Error]);
```
Reject the promise by switching its status from `NEW` to `REJECTED` and storing the error.

parameters:

`error` the error to be stored in the promise as the error that stops the execution.

### new

```
function new (): SrdRef[Promise[ResultType]];
```
Creates a new promise with a status of `new`.

return value:

A reference to the promise of the specified type.

### then

```
handler [ThenType: type] this.then(
    callback: closure (input: ResultType, promise: ref[Promise[ThenType]])
): SrdRef[Promise[ThenType]];
```
A template method to determine what needs to be executed after this promise is resolved.

parameters:

`callback` a closure  to be called when the promise is resolved.

return value:

A reference to a promise with the specified result type.

### catch

```
handler this.catch(
    callback: closure (err: SrdRef[Error], promise: ref[Promise[ResultType]])
): SrdRef[Promise[ResultType]];
```
A method to determine what needs to be executed when an error occurred (when the promise is rejected).

parameters:

`callback` a closure to be called when the promsie is rejected.

return value:

A reference to a promise with the specified result type.

### all

```
function all (inputs: Array[SrdRef[Promise[ResultType]]]): SrdRef[Promise[Array[ResultType]]];
```
Specify a set of promises, so that we consider the promise resolved when all of those promises are resolved,
and rejected when at least one of them is rejected.

parameters:

`inputs` a set of promises with the same result type.

return value:

A reference to a promise with the specified result type.

### ignoreResult

```
handler this.ignoreResult(): SrdRef[Promise[Int]];
```
Ignore promise's result.

This method can be used to ignore the result while using the `all` function, when the set of promises provied to `all`
do not have the same result type, since `all` function needs a set of promises with the same type, otherwise, the
compiler will give us an error.

return value:

A reference to a promise with an `Int` result type, and a result equal to `0`. This integer result has no meaning
and is only used because using the type of Void was not straight forward.

### Status

```
def Status: {
    def NEW: 0;
    def RESOLVED: 1;
    def REJECTED: 2;
}
```
This three states of a promsie.

## Recursive Promises

In some cases you need to create recursive promises that makes a very long or an infinite loop. An example
of these are promsies that periodically read data from the net or promises that runs periodically with
a delay between runs or promises that catch errors while reading data from the web and wants to
indefinitely keep trying until the server issue is resolved. In such cases if we use the `resolve` method
to resolve the promise with a new child promise we can end up with a stack overflow in addition to
consuming more memory than actually necessary.

To solve this the Promises library provides a `retry` method. This method is similar to `resolve` except
that it reuses the currnt promise to avoid growing the stack or the memory footprint as it replaces
the child promise with a new one rather than chaining the new promise to the existing chain. To clarify
this assume you have promise p1 with a `then` operation applied to it resulting in a t1 object that
depends on p1. If you want to recursively retry the operation inside t1 you'll need to create p2
and apply `then` to it to get t2 and use it in `resolve`. After five iterations you'll end up with
the following timeline for dependency updates:

* t1 -> p1
* t1 -> t2 -> p2
* t1 -> t2 -> t3 -> p3
* t1 -> t2 -> t3 -> t4 -> p4
* t1 -> t2 -> t3 -> t4 -> t5 -> p5

If instead we use the `retry` method then we don't need to apply `then` on the new promises we
are creating, instead we just pass the new promise to the existing `then` object to reuse it,
which results in the following timeline:

* t1 -> p1
* t1 -> p2
* t1 -> p3
* t1 -> p4
* t1 -> p5

The `retry` operation can be used in `then` and `catch`:

### ThenPromise

To use `retry` inside the closure of `then` we need to update the second arg of the closure
from `Promise` to `ThenPromise` and then use `retry` on the promise, as in the following
example:

```
def promise: SrdRef[Promise[Num]] = Promise[Num].new();
def then: SrdRef[Promise[String]] = promise.then[String](
    closure (input: Num, p: ref[ThenPromise[String, Num]]) {
        Console.print("ThenPromise triggered %d\n", input.val);
        promise = Promise[Num].new();
        p.retry(promise);
    }
);
def i: Int;
for i = 0, i < 10, ++i {
    promise.resolve(Num(i));
}
```

### CatchPromise

To use `retry` inside the closure of `catch` we need to update the second arg of the closure
from `Promise` to `CatchPromise` and then use `retry` on the promise, as in the following
example:

```
def promise: SrdRef[Promise[Num]] = Promise[Num].new();
def catch: SrdRef[Promise[Num]] = promise.catch(
    closure (err: SrdRef[Error], p: ref[CatchPromise[Num]]) {
        Console.print("CatchPromise triggered. error: %s\n", err.getMessage().buf);
        promise = Promise[Num].new();
        p.retry(promise);
    }
);
def i: Int;
for i = 0, i < 10, ++i {
    promise.reject(castSrdRef[SrdRef[GenericError]().{
        construct();
        code = String("example_err") + i;
        message = String("Unknown error ") + i;
    }, Error]);
}
```

