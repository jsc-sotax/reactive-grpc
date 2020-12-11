# RxJS wrapper for gRPC and TypeScript

A package that wraps the callback and event based [official gRPC implementation](https://www.npmjs.com/package/@grpc/grpc-js) with Promises and RxJS Observables.
Works well in conjunction with [grpc-tools](https://www.npmjs.com/package/grpc-tools) and [grpc_tools_node_protoc_ts](https://www.npmjs.com/package/grpc_tools_node_protoc_ts).

#### Before
```typescript
class ExampleService implements IExampleServer {
  incrementStream(call: grpc.ServerDuplexStream<OneNumber, OneNumber>): void {
    call.on("data", (number) => {
      call.write(new OneNumber().setA(number.getA() + 1));
    });
    call.on("end", () => call.end());
  }
}
```

#### After
```typescript
defineService<IExampleServer>((serviceGrpcPb as any)["Example"], {
  incrementStream(request: Observable<OneNumber>): Observable<OneNumber> {
    return request.pipe(map((number) => new OneNumber().setA(number.getA() + 1)));
  },
});
```

## Installation

`reactive-grpc` uses `rxjs` and `@grpc/grpc-js` as peer dependencies. As such, the desired package versions must be installed alongside `reactive-grpc`.

### yarn
```bash
yarn add reactive-grpc rxjs @grpc/grpc-js
```

### npm
```bash
npm install --save reactive-grpc rxjs @grpc/grpc-js
```

## Usage

### Server implementation
You can either "reactify" individual service methods or entire services. The latter is recommended, due to better type inference.

#### Reactify an entire service (recommended)
Use the `defineService` function to convert an object implementing the reactive service interface into a regular gRPC service. The reactive type is created automatically from the regular interface generated by `grpc-tools`.
```typescript
import { interval, Observable } from "rxjs";
import { map, reduce } from "rxjs/operators";

import { defineService } from "reactive-grpc";

// Generated by grpc-tools
import { OneNumber, TwoNumbers, Empty } from "../generated/service_pb";
import { IExampleServer } from "../generated/service_grpc_pb";
import * as serviceGrpcPb from "../generated/service_grpc_pb";

export default defineService<IExampleServer>((serviceGrpcPb as any)["Example"], {
  async addTwoNumbers(request: TwoNumbers): Promise<OneNumber> {
    return new OneNumber().setA(request.getA() + request.getB());
  },
  addStreamOfNumbers(request: Observable<OneNumber>): Promise<OneNumber> {
    return request
      .pipe(
        reduce((acc, value) => acc + value.getA(), 0),
        map((value) => new OneNumber().setA(value))
      )
      .toPromise();
  },
  getFibonacciSequence(request: Empty): Observable<OneNumber> {
    let a = 0;
    let b = 1;
    return interval(100).pipe(
      map(() => {
        let next = a + b;
        a = b;
        b = next;
        return new OneNumber().setA(a);
      })
    );
  },
  runningAverage(request: Observable<OneNumber>): Observable<OneNumber> {
    let average = 0;
    return request.pipe(
      map((value, index) => {
        average = (value.getA() + index * average) / (index + 1);
        return new OneNumber().setA(average);
      })
    );
  }
});
```
If you require advanced functionality from the standard gRPC call objects, such as reading the metadata or watching the cancellation status, you may simply add the usual argument:
```typescript
export default defineService<IExampleServer>((serviceGrpcPb as any)["Example"], {
  runningAverage(
    request: Observable<OneNumber>,
    call: grpc.ServerDuplexStream<OneNumber, OneNumber>
  ): Observable<OneNumber> {
    ...
  },
}
```
Methods with `Promise<ResponseType>` return types can also return objects instead to include trailer metadata and flags, which would otherwise be provided when calling the `callback` function:
```typescript
export default defineService<IExampleServer>((serviceGrpcPb as any)["Example"], {
  async addTwoNumbers(request: TwoNumbers): Promise<OneNumber> {
    return {
      value: new OneNumber().setA(request.getA() + request.getB()),
      trailer: ...,
      flags: ...
    };
  },
}
```

#### Reactify individual methods
Use `defineUnaryMethod`, `defineRequestStreamMethod`, `defineResponseStreamMethod` and `defineBidirectionalStreamMethod` to wrap your reactive function definitions. You can then assign the returned function to, for example, a member function of a non-reactive gRPC service:
```typescript
import { interval, Observable } from "rxjs";
import { map, reduce } from "rxjs/operators";

import {
  defineUnaryMethod,
  defineRequestStreamMethod,
  defineResponseStreamMethod,
  defineBidirectionalStreamMethod,
} from "reactive-grpc";

import { OneNumber, TwoNumbers, Empty } from "../generated/service_pb";
import { IExampleServer } from "../generated/service_grpc_pb";

/** Server of the example service that wraps each method individually. */
export default class ExampleServer implements IExampleServer {
  addStreamOfNumbers = defineRequestStreamMethod(function (
    request: Observable<OneNumber>
  ): Promise<OneNumber> {
    return request
      .pipe(
        reduce((acc, value) => acc + value.getA(), 0),
        map((value) => new OneNumber().setA(value))
      )
      .toPromise();
  });
}
```

### Client
To create a reactive gRPC client, you must first create a regular client instance, and then apply `reactive-grpc`'s `reactifyClient` function:
```typescript
import * as grpc from "@grpc/grpc-js";
import * as serviceGrpcPb from "../generated/service_grpc_pb"; // Generated by grpc-tools

import { reactifyClient } from "reactive-grpc";

const ExampleClient = grpc.makeClientConstructor(
  (serviceGrpcPb as any)["Example"],
  "ExampleService"
);

const client = (new ExampleClient(
  port,
  grpc.credentials.createInsecure()
) as unknown) as serviceGrpcPb.ExampleClient;

const reactiveClient = reactifyClient(
  (serviceGrpcPb as any)["Example"],
  client
);
```
The new client then creates an API with Promises and Observables:
```typescript
try {
  const sum = await reactiveClient.addNumbers(
    from([1, 2, 3, 4]).pipe(map((num) => new OneNumber().setA(num))),
  );
  console.log(sum.getA());
} catch (err) {
  // Handle gRPC errors here.
}
```
```typescript
const observable = reactiveClient.getFibonacciNumbers(new OneNumber().setA(20));
observable.subscribe(
  (value) => console.log(value.getA()),
  (err) => console.log("Oh no!"), // Handle gRPC errors here.
  () => console.log("Done!")
);
```
Optionally, you can also supply Metadata and CallOptions as the second and third parameters of the calls.
To retrieve the standard gRPC call object, use `.call` on the returned Promise or Observable:
```typescript
console.log(observable.call);
```

Note that unsubscribing from a server side stream will automatically cancel the request:
```typescript
const observable = reactiveClient.getFibonacciNumbers(new OneNumber().setA(20));
// Will produce a gRPC CANCELLED error after receiving 5 values.
// The error will be ignored by the client.
observable.pipe(take(5)).subscribe(
  (value) => console.log(value.getA()),
  (err) => console.log("Oh no!"),
  () => console.log("Done!")
);
```
