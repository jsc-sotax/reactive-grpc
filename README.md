# RxJS wrapper for gRPC and TypeScript

A package that wraps the callback and event based [official gRPC implementation](https://www.npmjs.com/package/grpc) with Promises and RxJS Observables.
Works well in conjunction with [grpc-tools](https://www.npmjs.com/package/grpc-tools) and [grpc_tools_node_protoc_ts](https://www.npmjs.com/package/grpc_tools_node_protoc_ts).

#### Before
```typescript
class ExampleService implements IExampleServer {
  incrementStream(call: grpc.ServerDuplexStream<OneNumber, OneNumber>): void {
    call.on("data", (number) => {
      call.write(new OneNumber().setA(number + 1));
    });
    call.on("end", () => call.end());
  }
}
```

#### After
```typescript
defineService<IExampleServer>(ExampleService, {
  runningAverage(request: Observable<OneNumber>): Observable<OneNumber> {
    return request.pipe(map((number) => new OneNumber().setA(number + 1)));
  },
});
```

## Installation

`reactive-grpc` uses `rxjs` and `grpc` as peer dependencies. As such, the desired package versions must be installed alongside `reactive-grpc`.

### yarn
```bash
yarn add reactive-grpc rxjs grpc
```

### npm
```bash
npm install --save reactive-grpc rxjs grpc
```

## Usage

### Server implementation
You can either "reactify" individual service methods or entire services. The latter is recommended, due to better type inference.

#### Reactify an entire service (recommended)

#### Reactify individual methods
Use `defineUnaryMethod`, `defineRequestStreamMethod`, `defineResponseStreamMethod` and `defineBidirectionalStreamMethod` to wrap your reactive function definitions. You can then assign the returned function to, for example, a member function of a non-reactive gRPC service:
```typescript
import { Observable } from "rxjs";
import { map, reduce } from "rxjs/operators";

import {
  defineUnaryMethod,
  defineRequestStreamMethod,
  defineResponseStreamMethod,
  defineBidirectionalStreamMethod,
} from "reactive-grpc";

import { OneNumber } from "../generated/service_pb"; // Generated by grpc-tools
import { IExampleServer } from "./generated/service_grpc_pb"; // Generated by grpc-tools

class ExampleService implements IExampleServer {
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
To create a reactive gRPC client, you must first create a regular client instance, and then apply `reactive-grpc`s `reactifyClient` function:
```typescript
import * as grpc from "grpc";
import { reactifyClient } from "reactive-grpc";
import { ExampleClient, ExampleService } from "./generated/service_grpc_pb"; // Generated by grpc-tools

const regularClient = new ExampleClient("localhost:5000", grpc.credentials.createInsecure());
const reactiveClient = reactifyClient(ExampleService, regularClient);
```
The new client then creates an API with Promises and Observables:
```typescript
try {
  const sum = await reactiveClient.addNumbers(
    from([1, 2, 3, 4]).pipe(map((num) => new OneNumber().setA(num))),
  );
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