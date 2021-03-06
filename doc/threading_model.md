﻿# gRPC C++ Threading Model

## Introduction
This document serves as a introduction to the fundamentals of the gRPC threading model. gRPC generates service code in two fashions - synchronous and asynchronous API. These two closely correlate to the threading model in play. 

## Synchronous API Threading model
This is the easiest model to start with and could be a satisfactory model for a large number of services. As such, the implementation is efficient (and is not a ‘toy’ implementation). In this model, gRPC takes care of the threading completely and the application code need to worry only about handling of the actual rpc call.
* When the server starts up, a thread pool is created to handle incoming rpc requests. The default implementation of this thread pool is called DynamicThreadPool. It creates threads equal to number of processors in the system and all of them are in waiting state.
* DynamicThreadPool has in­built scaling mechanism to create additional threads on demand. For integrators interested in having a little more control on the behavior of the thread pool, they can define GRPC_CUSTOM_DEFAULT_THREAD_POOL and provide the implementation of the thread pool interface (which is literally just 1 function that adds work to the pool).
* As soon as enough data is available to process a complete rpc, the worker thread moves ahead with handling the rpc.
* **Pros**
	* Simple to use and get going.
	* Efficiency would be fine for most applications. 
* **Cons**
	* The application code in the service implementation could be called concurrently from multiple threads - for the same client or different clients. As a result, the service implementation must use proper synchronization when accessing shared state.
	* An entire worker thread is occupied until the rpc finishes. This may lead to scaling problems based on the use case.
	* If you have long running streaming rpcs, a thread would be occupied for it until it finishes, even if there are long gaps with no streaming messages. If you have multiple clients accessing the same rpc concurrently, a separate thread is spawned to serve each client.
	* Even unary rpcs could potentially occupy a thread for long time if the rpc handling requires blocking IO to an external entity (db/external service etc.).

## Asynchronous API Threading model
The asynchronous model is considerably difficult to work with but is the right tool of choice when finer control on the threading aspect of the rpc handling is desired. In this model, gRPC does not create any threads internally and instead relies on the application to handle the threading architecture. 
* The application tells gRPC that it is interested in handling an rpc and provides it a completion key(`void*`) for the event (a client making the aforementioned rpc request).
* The application then calls `Next` (potentially a blocking call) on completion queue waiting for a completion key to become available. Once the key is available, the application code may react to the rpc associated with that key by executing the code that it chooses. The rpc may require more completion keys to be added to the completion queue before finishing if it is a streaming rpc, since each read or write on the stream has its own completion event.
* **Pros**
	* Allows the application to bring in it’s own threading model.
	* No intrinsic scaling limitation. Provides best performance to the integrator who is willing to go the extra mile.
* **Cons**
	* The API needs considerable boilerplate/glue code to be implemented. 



