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
        code = 1;
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

### Error

```
class Error {
    handler this.getCode(): Int as_ptr;
    handler this.getMessage(): String as_ptr;
}
```
This class holds the error's information.

`getCode` return the error's code.

`getMessage` return the error's message.

This is an abstract class.

### GenericError

```
class GenericError {
    @injection def error: Error;
    def code: Int;
    def message: String;
    handler (this: Error).getCode(): Int set_ptr;
    handler (this: Error).getMessage(): String set_ptr;
}
```

This class is an implementation of the abstract class `Error` that allows storing arbitrary code and message.

