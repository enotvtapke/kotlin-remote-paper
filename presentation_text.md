---
marp: true
paginate: true
---

Good day. My name is Aleksandr Stupnikov. The topic of my master thesis is "Lightweight RPC for Kotlin Multiplatform".

---

Kotlin is the languages that is vastly used for building software that communicates over the network. Such as backend microservices, client-server applications, distributed systems, etc. In all of these, the application code naturally splits into two parts. The first is business logic. It is unique, high-value part specific to each application. The second part is network code like routing, serialization, HTTP client setup, error handling. This network code is largely repetitive across applications. In other words it is boilerplate. Ideally, the network part should be as small as possible so that developers can focus on business logic. But with current Kotlin tools, the network part remains quite large. Let's look at why.

---

There are two main technologies used for network communication in Kotlin today. First is Ktor — a general-purpose HTTP framework from JetBrains. Second is gRPC — Google's cross-language RPC framework. And there is also not popular but promising RPC framework from JetBrains called Kotlin RPC. Each of these frameworks has a different trade-off between flexibility and the amount of boilerplate required. Let me show the specific problems with each.

---

With Ktor, the developer must manually write routing logic — specifying URL paths and HTTP methods. Developer also must handle serialization and deserialization of request and response bodies, write HTTP client calls and manage error handling. This gives fine-grained control, which is valuable when you need it — for example, for complex REST APIs with custom headers. But for many use cases, especially internal service-to-service communication, this level of control is unnecessary, and the boilerplate is pure overhead.

---

gRPC and Kotlin RPC take a different approach. They follow the Remote Method Invocation pattern. Developer define a service interface, implement it in a class, and then create a client proxy for that interface. This does reduce boilerplate compared to raw Ktor. However, there are significant drawbacks. First, developer must declare and implement an interface just to make functions remotely callable. Second, top-level functions cannot be called remotely at all — everything must live inside a service. Third, at the call site, a remote call looks identical to a local call. There is nothing that tells developer that this is a network operation that might fail or be slow. These are the problems we set out to solve.

---

The goal of this thesis is to develop an RPC framework for Kotlin Multiplatform that reduces boilerplate compared to existing approaches. The objectives are structured as follows. First, prototype an RPC framework that uses context parameters as the mechanism for expressing remote calls. I will explain context parameters later. Second, prototype support for distributed objects — remote classes whose state lives on the server. Third, implement and test the framework as a Kotlin compiler plugin with a runtime library. Fourth, evaluate the result by comparing it with existing alternatives — Ktor, gRPC, and Kotlin RPC — in terms of boilerplate and developer experience.

---

Now, the specific technical requirements the framework must satisfy. First, enable calling any named Kotlin function remotely — not just methods inside service interfaces. This means top-level functions, extension functions, class methods should all be callable. Second, distinguish remote functions from local ones at the type level, so that the developer always knows when a network call is happening — this addresses the invisibility problem we saw with gRPC and Kotlin RPC. Third, preserve Kotlin Multiplatform compatibility, so the framework works on all targets — JVM, JavaScript, Native, and WebAssembly — without relying on reflection.

---

Before describing the solution, let me introduce the theoretical foundation. Coeffects are a type-system mechanism for tracking how computations depend on their environment. They are the dual of effects: while effects describe what a computation does — like writing to a file or throwing an exception — coeffects describe what a computation requires from its environment. In theory, a wide subset of coeffects are modeled using indexed comonads. In practice, narrower subset of coeffects can be expressed as implicit parameters — values that are passed to a function automatically from the surrounding context, rather than explicitly by the caller. The key insight is: a remote function is basically a function with coeffect, because it requires from an environment a way to execute code on a remote machine. This requirement is a coeffect.

---

This idea is not new and is used in a world of functional languages. For example, in Haskell library Servant remote procedures return values in the ClientM monad. ClientM is essentially a Reader monad — it implicitly carries the remote server configuration and describes a coeffect. On a slide hello and position functions result in a remote call, and it is clear from their return value type.

Notice that there are no service interfaces, top-level functions work perfectly well, and the return type ClientM makes it explicit that this is a remote call. This is exactly the API ergonomics we want. The question is how to bring this to Kotlin — a language without monads.

---

The answer is Kotlin's experimental feature called context parameters. Context parameters are basically implicit parameters — they are resolved automatically from the enclosing scope, without the caller having to pass them explicitly. It means that they can express a coeffect. In our framework, a remote function declares a context parameter of type RemoteContext. This RemoteContext determines where the function executes — locally or on a remote server. It also describes how to execute function in case it executes remotely. RemoteContext is resolved implicitly at the call site, just like the Reader monad in Haskell. And makes remote calls visible at the type level. This brings the extended functional RPC approach to Kotlin without using constructions foreign to Kotlin (like monads).

---

Let me explain RemoteContext in details because it lies in the heart of the framework. RemoteContext is a sealed interface parameterized by a RemoteConfig type. It has two implementations. LocalContext is a singleton — it indicates that the function should execute its original body locally. ConfiguredContext wraps an actual remote configuration — a server URL, an HTTP client, and so on. Since LocalContext extends RemoteContext<Nothing> and Nothing is Kotlin's bottom type LocalContext is a subtype of RemoteContext<T> for any T. This means any remote function can always be called in a local context — it always type-checks. On the server side, functions are invoked in LocalContext, so the original body executes directly without any network call.

---

Here is the developer experience. To make a function remote, developer annotates it with @Remote annotation and adds a context parameter of type RemoteContext. The function body is the actual implementation — just normal Kotlin code. On the client side, user open a context block with a ConfiguredContext that contains the server configuration. Inside that block, user calls multiply as if it were a local function. Under the hood, this call goes over the network. On the server side — inside the framework's internals — the same function is called within LocalContext, and the original body executes directly. The developer writes the function once, and it works in both contexts.

---

The magic happens in a Kotlin compiler plugin. When the plugin sees the @Remote annotation, it transforms the function body at the level of intermediate representation. The original body is wrapped in a conditional. If the context is LocalContext, the original body runs — this is the server path. Otherwise, the context must be a ConfiguredContext, and the plugin generates code that packages the function name and arguments into a RemoteCall object and sends it to the server using the client from the configuration. This transformation is completely automatic. Developer writes a normal function, and the plugin handles the dual execution semantics.

---

Supporting Kotlin Multiplatform introduces a specific challenge. On non-JVM platforms — JavaScript, Native, WebAssembly — there is no reflection. This means that program cannot discover information about itself at runtime, but our server needs to know which remote functions exist and how to invoke them. Our solution: the compiler plugin collects metadata about all remote functions at compile time. The developer calls an intrinsic function called genCallableMap(), and the compiler plugin replaces that call with a map containing entries for every @Remote function — their names, parameter types, return types, and an invocator lambda that calls the actual function. Then user only needs to pass generated map when creating a server. No reflection needed. This works on all Kotlin Multiplatform targets. This same mechanism is also used for serialization on both client and server.

---

Beyond standalone functions, the framework supports distributed objects. A class annotated with @RemoteSerializable can have remote methods. In this example, Calculator holds mutable state on the server. The companion object's invoke operator is also remote — so calling Calculator(5) on the client actually creates the object on the server and returns a lightweight stub to the client. Subsequent method calls like multiply(6) are forwarded to the server, where they operate on the real object with its preserved state. From the client's perspective, it looks like a normal object, but the state lives entirely on the server.

---

Under the hood, when a remote object is created on the server, the real instance is stored in a RemoteInstancesPool, keyed by a unique identifier. The client receives a stub — a generated subclass that contains only the ID and the server URL. When a method is called on the stub, it triggers a remote call. The server looks up the real instance by ID and executes the method on it. For garbage collection, we use a lease-based mechanism. The client periodically renews leases for the stubs it holds. When a lease expires — meaning the client no longer needs the object — the server removes the instance from the pool. On the client side, we use weak references to detect when the application drops a stub.

---

It is worth noticing that our framework is transport-agnostic by design — the core framework does not depend on any specific  library to make network requests. However, the core-ktor module provides an out of the box integration with Ktor. Here you can see a typical server setup. The KRemote Ktor plugin is installed with the callable map. Then, inside the routing block, you can mix remote endpoints with standard Ktor features. In this example, one endpoint is protected with basic authentication, while another is open. Authentication, logging, CORS, rate limiting — everything from the Ktor ecosystem works alongside Kotlin Remote with no special configuration.

---

Let's compare what a developer must write for each new remote function. With gRPC, each function requires an IDL definition for the message and the RPC method as well as an override in the implementation class, and conversion functions between protobuf and domain types. With Kotlin RPC, each function requires a signature in the @Rpc interface and an override in the implementation class. With our framework called Kotlin Remote, user just adds the @Remote annotation and a context parameter to the function itself. That is the entire per-function overhead.

---

The framework has several limitations that are important to acknowledge. It is optimized for a shared-codebase model, where client and server are compiled together and have access to the same remote function. However, it is possible to completely separate client and server code just with a little more boilerplate. Another limitation is that there is no cross-language support — both sides must be Kotlin. On top of that all parameters and return values of remote functions must be serializable (of course). The last limitation is that the coroutine context is not preserved across the remote boundary.

---

To summarize. We developed Kotlin Remote — an RPC framework that uses context parameters as coeffects to eliminate the need for service interfaces. All four objectives were completed: any named Kotlin function can be called remotely, remote calls are visible at the type level, distributed objects are supported, and the framework is fully compatible with Kotlin Multiplatform. Thank you for your attention. I am ready for questions.
