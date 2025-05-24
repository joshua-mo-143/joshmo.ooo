+++
title = "Exploring and (minimally) recreating Axum"
description = "Let's learn how the Rust web framework Axum is made by analysing its core components, then re-creating it."
template = "blog-page.html"
tags = ["axum", "tutorial"]

[extra]
thumb = "exploring-and-recreating-axum-thumb.png"
+++

## Introduction
Hi! In this article we will try to understand fully how Axum works at the base level, as well as discussing some of the design choices and overall architecture. We will also learn a bit about how it works in practice by writing our own little mini-Axum (the Axum equivalent of Tokio's mini-Redis, if you will).

Our mini-Axum will, by the end of the article, be able to do the following:
- Take requests and return handlers based on HTTP request method
- Store and use state
- Use the extractor pattern to make it easy to add args to our handlers
- Implement a flexible response type via traits
- Be able to use middleware (to a degree)

If you just want to check out the MVP I've made for this article, you can do that [here.](https://github.com/joshua-mo-143/mini-axum) However, I would suggest reading the article beforehand as it is likely to provide some much needed clarity on some aspects of the implementation.

This will be the first post in what will hopefully be a series of posts where we explore the internals of different crates and understanding how they work, and hopefully becoming better engineers in the process and have some lessons that we can take away to things that we're all working on. No promises, though!

While an attempt will be made to create high quality code where possible and the code **does** compile (I wrote all of it by hand!), do bear in mind that some of the code is quite experimental and as such you may come across some caveats should you decide to run some personal experimentation yourself. An effort has been made to record them at the end of the article.

Without further ado, let's get into it! Be prepared - this is a relatively long read.
## What is Axum?
Axum is a HTTP web framework by the Tokio team, who are currently maintaining a non-trivial amount of crates related to the async Rust ecosystem.

Its primary objective is to be modular, flexible and easy to use. It requires no macros, using purely function pointers as the handlers. You can declaratively parse requests using extractors (we'll get into this later), with simple and predictable error handling. You can also additionally generate responses with minimal boilerplate.

One thing that sets Axum apart from other frameworks is that it has, by default, full compatibility with the `tower` crate and ecosystem. This allows axum to be much leaner in terms of not needing to re-implement a full middleware system itself.

It is currently split into a few crates:
- `axum` - the crate itself that we, the public, use
- `axum-extra` - a crate for extra stuff that is helpful for Axum users, but not necessary for the core DX
- `axum-macros` - a crate for Axum macros, one of which is a debugging macro that's very useful for debugging routes, etc (we'll get into this later on)
- `axum-core` (the core set of traits and other related core utilities for Axum)

This is par the course for most relatively large Rust codebases: crates are split by domain or purpose, which reduces compile time (via reducing bloat) and allows core crates to be much more lean.

## How does Axum work under the hood?
Before we actually start building anything, let's have a quick look at how the sausage is made, so to speak. We'll discuss all the core components and how they contribute to the Axum framework.
### Hyper & Tower as the core of Axum
Axum, for the most part, is primarily a thin layer over `hyper` and `tower`. Tower is essentially a library for creating robust clients and servers that uses functional composition as its core design pattern to be process requests and return a response. By using `tower`, `axum` can create manipulate its services to be as composable as possible by allowing many different forms of service layering (`Route`, `Router`, `MethodRouter`... the list goes on).

Of course, the pipeline operations are primarily arbitrary and can do essentially whatever you want. In this case however, Axum takes in what you could essentially consider a `Vec<u8>` (a byte array), parses data from the request via the extractor pattern, then returns a type-compliant response.

Axum additionally relies upon `hyper`, a battle-tested Rust HTTP framework library for creating clients and servers. Using `hyper` allows Axum to not need to manually parse HTTP requests. In production, it's important to keep crates with differing purposes apart - not only does it allow for better abstractions down the road, it also reduces compile time.

### Routing
Generally speaking, routing in Axum on the user end is quite simple. You can have routes like this that require zero macro usage:

```rust
Router::new().route("/", get(hello_world)).route("/echo", post(echo_message))
```

Essentially, how this works is that using type wizardry and function pointers you can create route handlers where users can insert an arbitrary number of arguments that implement a given trait into the arguments (which parse data from a given request body/headers), and as long as the return type resolves to a HTTP response, you should be good to go.

From the internal codebase, this is what it roughly looks like:
```rust

// https://github.com/tokio-rs/axum/blob/main/axum/src/routing/mod.rs#L71
impl<S> Router<S> {
	// .. other methods
    pub fn route(self, path: &str, method_router: MethodRouter<S>) -> Self {
        tap_inner!(self, mut this => {
            panic_on_err!(this.path_router.route(path, method_router));
        })
    }
}

// https://github.com/tokio-rs/axum/blob/main/axum/src/routing/method_routing.rs#L547
pub struct MethodRouter<S = (), E = Infallible> {
    get: MethodEndpoint<S, E>,
    head: MethodEndpoint<S, E>,
    delete: MethodEndpoint<S, E>,
    options: MethodEndpoint<S, E>,
    patch: MethodEndpoint<S, E>,
    post: MethodEndpoint<S, E>,
    put: MethodEndpoint<S, E>,
    trace: MethodEndpoint<S, E>,
    connect: MethodEndpoint<S, E>,
    fallback: Fallback<S, E>,
    allow_header: AllowHeader,
}
```

On top of this, some methods are also provided to allow `tower` services to be created from this router type. Conveniently, there is also a method that allows for calling with state (depending on what the state is, basically):
```rust

// https://github.com/tokio-rs/axum/blob/main/axum/src/routing/method_routing.rs#L1286
impl<B, E> Service<Request<B>> for MethodRouter<(), E>
where
    B: HttpBody<Data = Bytes> + Send + 'static,
    B::Error: Into<BoxError>,
{
    type Response = Response;
    type Error = E;
    type Future = RouteFuture<E>;

    // required method for tower services
    // as axum doesn't need to handle this manually, this can typically
    // be safely ignored
    #[inline]
    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    #[inline]
    fn call(&mut self, req: Request<B>) -> Self::Future {
        let req = req.map(Body::new);
        self.call_with_state(req, ())
    }
}

```

Essentially, the method router will chain the routes together and try to find which one matches the request and invoke the relevant service. You can see below that a declarative macro has been set up to do exactly this:
```rust

// https://github.com/tokio-rs/axum/blob/main/axum/src/routing/method_routing.rs#L1118
impl<S, E> MethodRouter<S, E> {
	// .. other methods

    pub(crate) fn call_with_state(&self, req: Request, state: S) -> RouteFuture<E> {
        // set up a macro to automate boilerplate generation
        macro_rules! call {
            (
                $req:expr,
                $method_variant:ident,
                $svc:expr
            ) => {
                if *req.method() == Method::$method_variant {
                    match $svc {
                        MethodEndpoint::None => {}
                        MethodEndpoint::Route(route) => {
                            return route.clone().oneshot_inner_owned($req);
                        }
                        MethodEndpoint::BoxedHandler(handler) => {
                            let route = handler.clone().into_route(state);
                            return route.oneshot_inner_owned($req);
                        }
                    }
                }
            };
        }

        // written with a pattern match like this to ensure we call all routes
        let Self {
            get,
            head,
            delete,
            options,
            patch,
            post,
            put,
            trace,
            connect,
            fallback,
            allow_header,
        } = self;

        call!(req, HEAD, head);
        call!(req, HEAD, get);
        call!(req, GET, get);
        call!(req, POST, post);
        call!(req, OPTIONS, options);
        call!(req, PATCH, patch);
        call!(req, PUT, put);
        call!(req, DELETE, delete);
        call!(req, TRACE, trace);
        call!(req, CONNECT, connect);

        let future = fallback.clone().call_with_state(req, state);

        match allow_header {
            AllowHeader::None => future.allow_header(Bytes::new()),
            AllowHeader::Skip => future,
            AllowHeader::Bytes(allow_header) => future.allow_header(allow_header.clone().freeze()),
	        }
	    }
	}
}

```

From the above, you can see that essentially it just makes a method router that allows for several methods. On receiving a request, it will then try to go through each HTTP request method and check for a match (and call the function pointer if it does match). See the below for an example of how this looks like on the user end:

```rust

use axum::{Router, routing::{get, post}};

Router::new().route("/",
    // you can see here the handlers are essentially chained here
	get(hello_world).post(echo_message)
)

```

Axum uses [several macros](https://github.com/tokio-rs/axum/blob/dd8d4a47cb674f71d2163518a69568898f5e9077/axum/src/routing/method_routing.rs#L27) to be able to generate all the code for all of the given request methods without taking up what would otherwise be an egregious amount of boilerplate. While this does increase compile time slightly, I would argue that it actually makes sense to use macros here given that a lot of the code is very repetitive and there's no real justification otherwise to *not* use macros (let's face it, would *you* want all of that boilerplate in your library?).

In terms of matching routes, Axum primarily uses its own router. However, for path routing  it leverages [`matchit`](https://docs.rs/matchit/latest/matchit/) which is a high-performance, zero-copy URL router.  Primarily speaking, the value from this is that it lets `matchit` handle the URL parameters rather than trying to re-implement the same thing in Axum.
## Extractors
One of the core principles of Axum (and in fact a lot of other web frameworks) is that they make heavy use of the extractor pattern to be able to create ergonomic DX for routing. If you're interested in a full video talk, [Rob Ede from Kraken has a great talk on exactly this.](https://www.youtube.com/watch?v=7DOYtnCXucw)

In practice, extractors in Axum look like this:
```rust

use axum::Json;

async fn do_something(Json(json): Json<serde_json::Value>) -> &'static str {
	// ...
}
```

In terms of how extractors work, a basic implementation would look something like this:
```rust
// a mock trait to illustrate a type that can be derived from a byte array
trait FromRequest;

// a mock trait to illustrate a handler
trait Handler;

// a mock trait to illustrate a compliant return type
trait Response;

impl<T, F, R> Handler for Fn(T) -> F where
	T: FromRequest,
	F: std::future::Future<Output = R>,
	R: Response
	{
	// .. your methods and whatnot go here
}

```

This is of course pseudo-code, but you get the idea. You write a function pointer that takes one or more types that implements `FromRequest` (ie it can be derived from a request), processes it and returns a type that implements `Response`. Nothing too crazy. Of course, this *does* lead to a lot of boilerplate, which has evidently led to the use of another declarative macro, named simply (or perhaps amusingly) `all_the_tuples`:

```rust

// https://github.com/tokio-rs/axum/blob/main/axum-core/src/extract/tuple.rs#L74
all_the_tuples!(impl_from_request);

```

## Recreating Axum
Alright, so onto the fun part: recreating Axum. Get ready, because there's going to be a lot of information to digest. If you're re-creating this step-by-step, feel free to take breaks. I know I would.

To start, we're going to create our new project:

```bash

cargo init mini-axum
cd mini-axum

```

Next, we'll add our dependencies. Cue the long list of dependencies!

```bash

cargo add bytes futures http http-body-util hyper hyper-util \
serde serde-json tokio tower -F hyper/server,hyper-util/full,serde/derive,\
tokio/net,tower/util

```

What did we just add?
- `bytes`: A crate that provides an efficient container for storing and operating on contiguous memory slices. We add this crate because we need the import, but a noteworthy crate to use.
- `dashmap`: A highly concurrent hashmap. It's a straight upgrade from the regular hashmap in async scenarios if you have concurrency requirements.
- `futures`: A crate for dealing with futures. Required for imports.
- `http`: The HTTP crate. Required for imports.
- `http-body-util`: A noteworthy crate for converting `hyper` request bodies into a more conventional format we can use, like `Vec<u8>` (where necessary).
- `hyper`: The underlying server framework we're going to use.
- `hyper-util`: Utilities for `hyper`. In this case, we need the Tokio executor.
- `serde`: A crate for (de)serializing data. We use the `derive` feature to get convenient access to derives.
- `serde_json`: A companion crate for `serde` that lets you easily convert data to and from JSON.
- `tokio`: An async runtime for Rust - although if you're reading this, Tokio probably requires no introduction. Uses the `net` feature so we can get access to `tokio::net::TcpListener`.
- `tower`: A library for robust clients/servers. We add the `util` feature for access to `BoxCloneSyncService` (an extremely useful type/struct, which we'll get into later)

Let's get started!

### Creating our router/service
The first step on our journey will be creating the router/service itself. While I'd love to get into parsing HTTP manually, that itself could probably an article on its own - which is why we'll be leveraging `hyper`, a battle-tested library that powers several frameworks like Axum (of course!) and Rocket.

To be able to use `hyper` as a web server, we need to write a type that implements the `hyper::service::Service` trait - so we'll do exactly that. For now, our `Router` type is going to be a unit struct - we'll fill it out later.

```rust

use hyper::{body::Incoming, service::Service as HyperService};
use http::{Response, Request};
use http_body_util::Full;
use bytes::Bytes;

#[derive(Clone, Default)]
pub struct Router;

impl HyperService<Request<Incoming>> for Router
{
    type Response = Response<Full<Bytes>>;
    type Error = hyper::Error;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn call(&self, req: Request<Incoming>) -> Self::Future {
        Box::pin(async move { Ok(Response::builder().body(Full::new(Bytes::from("Hello world!"))))
        })
    }
}

```

As you can see, nothing too crazy. We return a box-pinned future (or a `BoxFuture`) that contains a `hyper` response.

The next thing to do is to create the `Service` struct by attaching together a `TcpListener` and our router (we'll assume that the user instantiates their own `TcpListener`).

```rust

pub struct Service {
    tcp: TcpListener,
    router: Router,
}

impl Service {
    pub fn new(tcp: TcpListener, router: Router) -> Self {
        Self { tcp, router }
    }

    async fn run(self) -> Result<(), Box<dyn std::error::Error>> {
        loop {
            let (stream, _) = self.tcp.accept().await?;
            let io = TokioIo::new(stream);

            let rtr = self.router.clone();
            tokio::task::spawn(async move {
                if let Err(err) = http1::Builder::new().serve_connection(io, rtr).await {
                    eprintln!("Error serving connection: {:?}", err);
                }
            });
        }
    }
}

```

Additionally, the Axum implementation of this actually also implements `IntoFuture`, allowing it to simply be awaited rather than needing to do anything else as this trait allows it to be turned into a future.

```rust

impl<S> IntoFuture for Service<S>
where
    S: Clone + Send + Sync + 'static,
{
    type Output = Result<(), Box<dyn std::error::Error>>;
    type IntoFuture = Pin<Box<dyn Future<Output = Self::Output> + Send>>;
    fn into_future(self) -> Self::IntoFuture {
        Box::pin(self.run())
    }
}

```

Now to actually try out our thing we just made. Create a new `examples` folder in your project root, then create a file called `basic.rs` - and put this code in:

```rust

use mini_axum::{Router, Service};
use serde_json::{Value, json};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let rtr = Router;
    let tcp = TcpListener::bind("127.0.0.1:9999")
        .await
        .expect("to be able to bind to localhost port 9999");

    let svc = Service::new(tcp, rtr);

    svc.await.unwrap();
}

```

Now if you run `cargo run --example basic` then visit `localhost:9999` in your browser, you should see `"Hello world!"` as the result. Your browser will additionally expect to be able to fetch a favicon, but that can be safely ignored as backend web servers tend to not need those unless you're serving HTML from your router.

### Response types
Before we can create any endpoints, we should probably define a response type. While `hyper`'s response type isn't particularly verbose, I would prefer to not have to write it over and over again as an end user.

Below is a short and simple example of this: we write a trait that allows types to be turned into our `MiniResponse` type, which then implements a method to turn it into a Hyper response.

```rust

use bytes::Bytes;
use http::{StatusCode, Response};
use http_body_util::Full;
use bytes::Bytes;

pub trait IntoMiniResponse {
    fn into_response(self) -> MiniResponse;
}

pub struct MiniResponse {
    code: StatusCode,
    content_type: String,
    bytes: Bytes,
}

impl MiniResponse {
    fn new(code: StatusCode, content_type: &str, bytes: Bytes) -> Self {
        Self {
            code,
            content_type: content_type.to_string(),
            bytes,
        }
    }

    pub fn hyper_response(self) -> Response<Full<Bytes>> {
        Response::builder()
            .status(self.code)
            .header("Content-Type", self.content_type)
            .body(Full::new(self.bytes))
            .unwrap()
    }
}

```

The rest of the work on this is primarily just writing `IntoMiniResponse` implementations for however many types you want to do it for. Here is one I wrote for returning JSON, as well as plain strings:

```rust

use serde::{Deserialize, Serialize};
use http::StatusCode;

#[derive(Deserialize, Serialize)]
pub struct Json<T>(pub T);

impl<T> IntoMiniResponse for (StatusCode, Json<T>)
where
    T: Serialize,
{
    fn into_response(self) -> MiniResponse {
        let (code, bytes) = self;
        let bytes = Bytes::from(serde_json::to_vec(&bytes.0).unwrap());

        MiniResponse::new(code, "application/json", bytes)
    }
}

impl<T> IntoMiniResponse for Json<T>
where
    T: Serialize,
{
    fn into_response(self) -> MiniResponse {
        let bytes = Bytes::from(serde_json::to_vec(&self.0).unwrap());

        MiniResponse::new(StatusCode::OK, "application/json", bytes)
    }
}


impl IntoMiniResponse for String {
    fn into_response(self) -> MiniResponse {
        let bytes = Bytes::from(&self);

        MiniResponse::new(StatusCode::OK, "text/plain", bytes)
    }
}

```

Since we own the `IntoMiniResponse` trait, we can also implement it for `Result<T, E>`:

```rust

impl<T, E> IntoMiniResponse for Result<T, E>
where
    T: IntoMiniResponse,
    E: IntoMiniResponse,
{
    fn into_response(self) -> MiniResponse {
        match self {
            Ok(res) => res.into_response(),
            Err(err) => err.into_response(),
        }
    }
}

```

Technically, we can't actually do anything useful with this yet. However when we get into writing our endpoints, we will need to have this functionality set up already as it'll be part of a required trait bound.
### Creating Axum-style endpoints
Here is where it starts to get a bit tricky. Buckle up!

As you probably already know, Axum takes function pointers as inputs for route handlers. The way this is managed is through layers of trait indirection. By converting your function pointers into structs, you can then ensure you don't get compiler errors from trait trickery as your types basically resolve to a concrete type which you can then serve from your router.

The below struct essentially represents a handler, with the input and state types converted to `PhantomData`, a zero-size value that can be used to essentially pretend that we have ownership of a type when nothing exists there.

```rust

pub struct IntoHandlerStruct<H, T, S> {
    inner: H,
	  state: S,
    _tytypes: PhantomData<T>,
}

pub trait IntoHandler<T, S>: Sized {
    fn into_handler(self, state: S) -> IntoHandlerStruct<Self, T, S>;
}

```

The next thing to do is to implement our new trait for `F` - that is to say, a function pointer that returns a future wrapping our response type:

```rust

impl<F, Fut, I, S> IntoHandler<(), S> for F
where
    F: Fn() -> Fut + Clone + Send + Sync + 'static,
    Fut: Future<Output = I> + Send + 'static,
    I: IntoMiniResponse,
{
    fn into_handler(self, state: S) -> IntoHandlerStruct<Self, (), S> {
        IntoHandlerStruct {
            inner: self,
            state,
            _tytypes: PhantomData,
        }
    }
}

```

There are a couple of things to note here:
- We shouldn't need to implement `IntoHandler` for a concrete type. The proper way to do this is to use a tuple for `T` (in `IntoHandler<T, S>`) - although in this case, there's no input types so we're using a unit `()` type instead.
- Although technically speaking state *is* implemented for every single function (by virtue of adding the state generic), this doesn't actually necessarily mean anything. For the most part, state in terms of handlers is just another extractor - and we'll be discussing a bit more in-depth on this later as well. I may have spent quite a few more hours than I should've trying to figure this out.

Once done, we then need to implement the `tower::Service` trait for `IntoHandlerStruct`
 which will allow it to be used with the `tower` ecosystem. Perfect for a composable router. Below are some notes on the type signatures so you don't get lost:
 - `F` represents a function that returns a future, which additionally implements the additional trait bounds
- `Fut` is the future itself. This trait indirection is required as `Future` is a trait.
- `I` is the response type. Not much to say about that.
- `S` is the state type. We're cloning state into handlers when they get called (and potentially sharing/sending state across threads), hence the trait bounds here.

```rust

impl<F, Fut, S, I> tower::Service<Request<Incoming>> for IntoHandlerStruct<F, (), S>
where
    F: Fn() -> Fut + Clone + Send + Sync + 'static,
    Fut: Future<Output = I> + Send + 'static,
    I: IntoMiniResponse,
    S: Clone + Send + Sync + 'static,
{
    type Error = hyper::Error;
    type Response = Response<Full<Bytes>>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;
    fn poll_ready(
        &mut self,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Result<(), Self::Error>> {
        std::task::Poll::Ready(Ok(()))
    }
    fn call(&mut self, _req: Request<Incoming>) -> Self::Future {
        let thing = self.inner.clone();

        Box::pin(async move { Ok((thing)().await.into_response().hyper_response()) })
    }
}

```

Next, we are going to use a struct called `tower::util::BoxCloneSyncService` - essentially, a generic boxed service that also implements Clone and Sync. This allows you to share and send it across threads, which is great for us as we're working in mostly a fully async scenario. Here is the type signature we'll be using:

```rust
type DynService = BoxCloneSyncService<Request<Incoming>, Response<Full<Bytes>>, hyper::Error>;
```

Now that we've defined our dynamic service type, we're going to use it in our router. We will also additionally define the `S` generic which will be our shared state. For now though, we will leave our router stateless and come back to it as we'll be covering it in another section later.

```rust

use std::sync::Arc;

// Note: Technically, if your RwLock ever needs to cross an await point, you should use tokio's RwLock.
// However, in our case, it doesn't, so we should be fine here.
use std::sync::RwLock;

#[derive(Clone, Default)]
pub struct Router<S = ()> {
    pub inner: Arc<RwLock<HashMap<String, DynService>>>,
    state: S
}

// stateless router - we'll come back to this!
impl Router<()> {
	fn new() -> Self {
	    Self {
            inner: Arc::new(RwLock::new(HashMap::new())),
            state: ()
        }
	}
}

```

In terms of adding a function to add a route, we need to set a pretty heavy amount of trait bounds. You can see the length of the type signature below is not particularly easy on the eyes:

```rust

impl<S> Router<S>
where
    S: Clone + Send + Sync + 'static,
{
    pub fn route<T, E>(self, route: &str, endpoint: E) -> Self
    where
        T: 'static + Sync + Send,
        E: IntoHandler<T, S> + Clone + Send + Sync + 'static,
        IntoHandlerStruct<E, T, S>: tower::Service<
                Request<Incoming>,
                Response = Response<Full<Bytes>>,
                Error = hyper::Error,
                Future = Pin<
                    Box<dyn Future<Output = Result<Response<Full<Bytes>>, hyper::Error>> + Send>,
                >,
            > + 'static,
    {
        let endpoint = endpoint.into_handler(self.state.clone());

        self.inner
            .write()
            .unwrap()
            .insert(route.to_string(), BoxCloneSyncService::new(endpoint));

        self
    }
}

```

The reason *why* we need the trait bounds in the first place is because we need to ensure that `T` is thread-safe (although we use collections of tuples for inputs, a tuple of any amount of elements can be considered 1 type). We also need to ensure that the input type itself also implements `IntoHandler` as we need to convert the function pointers into `IntoHandlerStruct`s.

What might be slightly more confusing however is setting trait bounds on `IntoHandlerStruct<E, T, S>`. While *we* know that this struct implements `tower::Service`, the compiler is unable to infer this by itself. Hence, we need to put the full type signature for the kind of `tower::Service` trait bounds we're using.

You may have noticed that while we're taking `IntoHandler<T, S>` for our route function, the type we're storing actually implements `Endpoint<S>`. While *we* know that the return type of `IntoHandler::into_handler()` returns `IntoHandlerStruct<E, T, S>`, the compiler doesn't - so we need to make sure we declare that in our function.

We will also additionally need to go back to our  `impl<S> Service<Request<Incoming>> for Router<S>` block and make some changes.

```rust

impl<S> Service<Request<Incoming>> for Router<S>
where
    S: Clone + Send + Sync + 'static,
{
    type Response = Response<Full<Bytes>>;
    type Error = hyper::Error;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn call(&self, req: Request<Incoming>) -> Self::Future {
	    // get the URI path and router
        let rdr = self.inner.read().unwrap();
        let path = req.uri().path();

		// clone state so we can put it in our router
        let state = self.state.clone();
        println!("Path: {path}");

		// if a path is found, run the resulting handler
		// if not, return 404
        if let Some(func) = rdr.get(req.uri().path()) {
            let mut func = func.clone();
            Box::pin(async move { func.call(req).await })
        } else {
            Box::pin(async move {
                Ok((StatusCode::NOT_FOUND, "Not found")
                    .into_response()
                    .hyper_response())
            })
        }
    }
}

```

Finally, let's go back to our `basic.rs` example we wrote earlier, and change it a bit. We will add a handler function that returns a JSON message and add it as a route in our router.

```rust

use hyper::StatusCode;
use mini_axum::{
    Router, Service,
    response::{IntoMiniResponse, Json},
};
use serde_json::{Value, json};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let rtr = Router::new()
        .route("/", hello_world);

    let tcp = TcpListener::bind("127.0.0.1:9999")
        .await
        .expect("to be able to bind to localhost port 9999");

    let svc = Service::new(tcp, rtr);

    svc.await.unwrap();
}

pub async fn hello_world() -> impl IntoMiniResponse {
    let json = json!({"message": "Hello world!"});

    (StatusCode::OK, Json(json))
}

```

Now this looks a bit more like the `axum` we're all used to!

If you run `cargo run --example basic` again and try visiting `localhost:9999` in your browser, you should get `{"message": "Hello world!"}` as a response with an option to prettify the returned JSON.

### Writing our own Axum-style extractor system
The process of using extractors is (in theory) relatively simple: you just add them as arguments to your handler, and it *just works*. For example (using the official Axum library):

```rust

use axum::Json;

async fn echo_message(Json(json): Json<serde_json::Value>) -> Json<serde_json::Value> {
	Json(json)
}

```

Not a particularly complex handler, mind you, but it should work if you were to insert this as a handler into an Axum server.

On the other hand, *implementing* extractors can be significantly more painful and tricky if you're not sure how to manipulate the type system. If you visited the extractor talk video I linked earlier, you probably have a good idea of what we're about to do.

The code for this part essentially relies upon two traits - `FromRequest` and `FromRequestParts` (we'll use the same naming here as there's no reason to deviate):
```rust

use http::Request;
use hyper::body::Incoming;

pub trait FromRequest<S>: Send + Sync {
    fn from_request(req: Request<Incoming>, state: &S) -> impl Future<Output = Self> + Send;
}

pub trait FromRequestParts<S>: Send + Sync {
    fn from_request_parts(
        req: http::request::Parts,
        state: &S,
    ) -> impl Future<Output = Self> + Send;
}

```

We will also want to implement `FromRequest` and `FromRequestParts` for types of our choice. For the demo we'll use JSON and State as State is basically already existent in our framework no matter what, and we've already implemented a JSON response.

```rust

impl<S, T> FromRequest<S> for Json<T>
where
    T: for<'a> serde::Deserialize<'a> + Send + Sync,
    S: Clone + Send + Sync,
{
    async fn from_request(req: Request<Incoming>, _state: &S) -> Self {
        let (_, body) = req.into_parts();
        let body: Vec<u8> = body.collect().await.unwrap().to_bytes().to_vec();
        let json: T = serde_json::from_slice(&body).unwrap();

        Json(json)
    }
}

pub struct State<T>(pub T);

impl<S> FromRequestParts<S> for State<S>
where
    S: Clone + Send + Sync,
{
    async fn from_request_parts(_req: Parts, state: &S) -> Self {
        State(state.to_owned())
    }
}

impl<S> FromRequest<S> for State<S>
where
    S: Clone + Send + Sync,
{
    async fn from_request(_req: Request<Incoming>, state: &S) -> Self {
        State(state.to_owned())
    }
}

```

Additionally, to remove code bloat we'll impl `FromRequest` for tuples of elements where the last element always implements `FromRequest`, but all other items implement `FromRequestParts`:

```rust

impl<S, T1> FromRequest<S> for (T1,)
where
    T1: FromRequest<S>,
    S: Clone + Send + Sync,
{
    async fn from_request(req: Request<Incoming>, state: &S) -> Self {
        let t1 = T1::from_request(req, &state).await;

        (t1,)
    }
}

impl<S, T1, T2> FromRequest<S> for (T1, T2)
where
    T1: FromRequestParts<S>,
    T2: FromRequest<S>,
    S: Clone + Send + Sync,
{
    async fn from_request(req: Request<Incoming>, state: &S) -> Self {
        let (parts, body) = req.into_parts();
        let t1 = T1::from_request_parts(parts.clone(), &state).await;

        let req = Request::from_parts(parts, body);
        let t2 = T2::from_request(req, &state).await;

        (t1, t2)
    }
}

```

Notes:
- Because each tuple is different here, it causes the implementations to be non-overlapping despite us not changing the `FromRequest` trait generic type whatsoever.
- Each request body can only be consumed **once** - the `FromRequest` trait consumes the body, hence why only one element is allowed to implement `FromRequest`.

Now to write the trait magic that makes it all work!

While the `axum` repository does use macros to minimise the amount of toil required to maintain the code for this, we will be implementing a much more minimal version to illustrate the point as after more than 1 or 2 arguments, it's mostly just copy and pasting.

As mentioned before when we wrote our original `IntoHandler` block, we implemented it for a function that essentially has no arguments. Here is the impl block for reference so you don't have to go halfway back up the blog post again:

```rust

impl<F, Fut, I, S> IntoHandler<(), S> for F
where
    F: Fn() -> Fut + Clone + Send + Sync + 'static,
    Fut: Future<Output = I> + Send + 'static,
    I: IntoMiniResponse,
{
    fn into_handler(self) -> IntoHandlerStruct<Self, (), S> {
        IntoHandlerStruct {
            inner: self,
            _tystate: PhantomData,
            _tytypes: PhantomData,
        }
    }
}

```

Now we'll be implementing it for a function pointer that *does* take arguments. If you're following along, it would be prudent to check that the first type of `IntoHandler` for your implementation uses a comma to turn the type into a tuple - if you use `(T1)` it will assume it's just any old generic and it won't compile.

```rust

// turn the function into a handler
impl<F, Fut, I, S, T1> IntoHandler<(T1,), S> for F
where
    F: Fn(T1) -> Fut + Clone + Send + Sync + 'static,
    Fut: Future<Output = I> + Send + 'static,
    I: IntoMiniResponse,
    T1: FromRequest<S> + Send + 'static,
{
    fn into_handler(self) -> IntoHandlerStruct<Self, (T1,), S> {
        IntoHandlerStruct {
            inner: self,
            _tystate: PhantomData,
            _tytypes: PhantomData,
        }
    }
}

impl<H, Fut, S, I, T1> tower::Service<Request<Incoming>> for IntoHandlerStruct<H, (T1,), S>
where
    H: Fn(T1) -> Fut + Clone + Send + Sync + 'static,
    Fut: Future<Output = I> + Send + 'static,
    I: IntoMiniResponse,
    T1: FromRequest<S> + Send + 'static,
    S: Send + Clone + Sync + 'static,
{
    type Error = hyper::Error;
    type Response = Response<Full<Bytes>>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;
    fn poll_ready(
        &mut self,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Result<(), Self::Error>> {
        std::task::Poll::Ready(Ok(()))
    }
    fn call(&mut self, req: Request<Incoming>) -> Self::Future {
        let thing = self.inner.clone();
        let state = self.state.clone();

        Box::pin(async move {
            let (t1) = T1::from_request(req, &state).await;
            Ok((thing)(t1).await.into_response().hyper_response())
        })
    }
}

```

While the generics are are somewhat intimidating, it's also mostly the same as the last time we impl'd `Endpoint<S>` and `IntoHandler<T, S>` except we are now adding trait generics that represent arguments to be taken by the function handler.

For functions that take two arguments, we need to ensure that the first argument implements `FromRequestParts` rather than `FromRequest` as we need to ensure the request body doesn't actually get consumed before the last argument consumes it. Below is what this would look like:

```rust

impl<F, Fut, I, S, T1, T2> IntoHandler<(T1, T2), S> for F
where
    F: Fn(T1, T2) -> Fut + Clone + Send + Sync + 'static,
    Fut: Future<Output = I> + Send + 'static,
    I: IntoMiniResponse,
    T1: FromRequestParts<S> + Send + 'static,
    T2: FromRequest<S> + Send + 'static,
{
    fn into_handler(self) -> IntoHandlerStruct<Self, (T1, T2), S> {
        IntoHandlerStruct {
            inner: self,
            _tystate: PhantomData,
            _tytypes: PhantomData,
        }
    }
}

impl<H, Fut, S, I, T1, T2> tower::Service<Request<Incoming>> for IntoHandlerStruct<H, (T1, T2), S>
where
    H: Fn(T1, T2) -> Fut + Clone + Send + Sync + 'static,
    Fut: Future<Output = I> + Send + 'static,
    I: IntoMiniResponse,
    T2: FromRequest<S> + Send + 'static,
    T1: FromRequestParts<S> + Send + 'static,
    S: Send + Clone + Sync + 'static,
{
    type Error = hyper::Error;
    type Response = Response<Full<Bytes>>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;
    fn poll_ready(
        &mut self,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Result<(), Self::Error>> {
        std::task::Poll::Ready(Ok(()))
    }
    fn call(&mut self, req: Request<Incoming>) -> Self::Future {
        let thing = self.inner.clone();
        let state = self.state.clone();

        Box::pin(async move {
            let (t1, t2) = <(T1, T2)>::from_request(req, &state).await;
            Ok((thing)(t1, t2).await.into_response().hyper_response())
        })
    }
}

```

For each tuple interpolation after this one, it's pretty much the same: just add the trait generics on and do the thing.

### Adding middleware
Now that we've implemented `tower::Service<T>` for our endpoints, we can now implement middleware for all of our services!

Due to extractors being such a present pattern in Rust frameworks, it is generally recommended to use middleware as more of a global add-on rather than using it for things like authz (for example) when you could use extractors instead.

Firstly, we'll create our own middleware layer that logs out the path and request method - then we'll add a `layer` method to our router.

A simple `LogLayer` middleware can be created like so that will print the HTTP method and URI, and then call the service:

```rust

use http::Request;
use hyper::body::Incoming;
use tower::Service;

#[derive(Clone)]
pub struct LogLayer;

impl<S> tower::Layer<S> for LogLayer {
    type Service = LogService<S>;

    fn layer(&self, inner: S) -> Self::Service {
        LogService { inner }
    }
}

#[derive(Clone)]
pub struct LogService<S> {
    inner: S,
}

impl<S> tower::Service<Request<Incoming>> for LogService<S>
where
    S: Service<Request<Incoming>> + Clone,
{
    type Error = S::Error;
    type Response = S::Response;
    type Future = S::Future;

    fn call(&mut self, req: Request<Incoming>) -> Self::Future {
        let (parts, body) = req.into_parts();
        let req = Request::from_parts(parts.clone(), body);

		println!("{method} {uri}", method = parts.method, uri = parts.uri);

        let mut service = self.inner.clone();

        service.call(req)
    }

    fn poll_ready(
        &mut self,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Result<(), Self::Error>> {
        std::task::Poll::Ready(Ok(()))
    }
}

```

Admittedly, this is not the most ergonomic way to do it, but it works. You may want to read [the current limitations of our implementation](#our-current-limitations) to explore why this is the case.

Next, to add a layering function for our router. Fortunately, this part is quite simple: we simply iterate over every service in our router, and layer it. To be able to do so requires putting a large amount of trait bounds on our `L` generic. However, as they primarily relate to the trait bounds of the `tower` services, you can guess without too much hesitation what exactly the trait bounds will be.

```rust

impl Router<S> where S: Clone + Send + Sync + 'static {
    // .. other fns
    pub fn layer<L>(mut self, layer: L) -> Self
    where
        L: Layer<DynService> + Clone + Send + Sync + 'static,
        L::Service: Service<
                Request<Incoming>,
                Response = Response<Full<Bytes>>,
                Error = hyper::Error,
                Future = BoxFuture<'static, Result<Response<Full<Bytes>>, hyper::Error>>,
            > + Clone
            + Send
            + Sync
            + 'static,
        <L::Service as Service<Request<Incoming>>>::Future: Send + 'static,
    {
        let res: HashMap<String, DynService> = self
            .inner
            .write()
            .unwrap()
            .clone()
            .into_iter()
            .map(|(k, v)| {
                let service = ServiceBuilder::new().layer(layer.clone()).service(v);

                (k, BoxCloneSyncService::new(service))
            })
            .collect();

        self.inner = Arc::new(RwLock::new(res));

        self
    }
}

```
### State in Axum
[State in Axum](https://docs.rs/axum/latest/axum/index.html#sharing-state-with-handlers) can be defined broadly as "some variables that are shared between handlers". In that sense, there are a total of 4 different ways that you can actually apply state to handlers:
- Injected from extractor patterns (the `State` extractor)
- Using `Extension<T>` layers which provide more flexibility but are less typesafe
- Using closure captures
- Using task local variables (via `tokio::task_local` macro)

Most people will likely opt for the first two options as a lot of the time using closure captures may not be ideal, and task-local variables rely on the task executor actually being able to use task-local variables and is more primarily for things like sharing state with things like `IntoResponse` implementations.

For reference, here is what an example of using State from an extractor looks like:

```rust

use axum::extract::State;
use axum::http::StatusCode;

#[derive(Clone)]
struct MyType;

// code in a function
let router = Router::new()
    .route("/", get(do_something))
    .with_state(MyType);

// our handler
async fn do_something(State(state): State<MyType>) -> StatusCode {
    // .. do something here

    StatusCode::OK
}

```

Pretty easy, all things considered. You can't change the state type (unless you use `FromRef`), but it's very ergonomic and mostly self-explanatory.

Of course, as you've probably noticed we have already implemented the first type of state which is built into the router endpoints. However, you can also implement `Extension<T>` as a layer by doing the following:
- Create an extension layer of type `T: Clone`. Implement `FromRequest<S>` and `FromRequestParts<S>` for it.
- Call `request.extensions_mut()` and insert the type in, then call the service as usual with the request that has had the extension data added.

Firstly, we'll create the layer type (so we can add it into our router):
```rust

struct Extension<T>(pub T);
struct ExtensionService<T, S> { ext: T, inner: S };

impl<S, T> Layer<S> for Extension<T> where T: Clone {
    type Service = ExtensionService<T>;

    fn layer(&self, inner: S) -> Self::Service {
        ExtensionService { ext: self.0.clone(), inner }
    }
}

impl<T, S> tower::Service<Request<Incoming>> for ExtensionService<T, S> where T: Clone, S: Service<Request<Incoming>> {
    type Error = S::Error;
    type Response = S::Response;
    type Future = S::Future;

    fn call(&mut self, mut req: Request<Incoming>) -> Self::Future {
        req.extensions_mut().insert(self.ext);

        let mut service = self.inner.clone();

        service.call(req)
    }

    fn poll_ready(
        &mut self,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Result<(), Self::Error>> {
        std::task::Poll::Ready(Ok(()))
    }
}
```

The second half of this is the `FromRequest` implementation. Pretty easy.
```rust

impl<T, S> FromRequest<S> for Extension<T> {
    async fn from_request(req: Request<Incoming>, state: &S) -> Self {
        let Some(val) = request.extensions.get::<T>() else {
            panic!("Welp, there's nothing here.");
        }

        Self(val)
    }
}
```

This will allow you to add extensions into your route handlers like so:

```rust

use mini_axum::{Extension};

// router code in a function
let rtr = Router::new().layer(Extension("Hello world!".to_string()));

async fn do_something(Extension(ext): Extension<String>) -> String {
    ext
}

```

Not entirely typesafe mind you as there may be issues with adding the extension data that can cause unwanted behaviour if you use the wrong type, but it's there.

## Finishing up
Thanks for reading! I hope this has been somewhat enlightening. This has been an extremely long post and I've spent many hours toiling and trying to understand the magic that makes Axum the framework that it is today, but we have done it. Hopefully, we have become better engineers for doing it. Below are additionally a list of current demo limitations and pitfalls to avoid while recreating the MVP should you decide to do so.

### Our current limitations
Of course, the demo we just created isn't without limitations/caveats. Here is a list of them below:
- Our `tower::Service` type creates pin-boxed futures rather than types that impl Future. This is not necessarily a problem by itself, but the way we've implemented `tower::Service` does not currently allow the default `tower` middlewares to be used with our system.
- We currently don't try to parse the HTTP method, although the more astute among you have probably recognised this already. To fix this, you can create a `MethodRouter` or some other struct that stores the routes (chaining together if needed) then adapting the router behaviour accordingly. This has been left as an exercise to the reader.
- There's no URL parameter parsing. You can re-implement this yourself with `matchit` and it should be relatively easy to do so.

### Potential pitfalls to avoid
While writing this article (and therefore the MVP demo), I've had my fair share of compiler persuasion to deal with and thought it might be a good idea to share the things I did wrong so you can avoid them in the future if you are embarking on a journey with a similar level of type trickery:
- Initially when I built my prototype, I had a trait called `Endpoint<S>` that was functionally the same as `tower::Service`. It *did* work, but trying to add middleware become a huge unfixable nightmare because of trait bounds. Eventually, I conceded to the compiler and ended up just integrating `tower` properly rather than trying to DIY it.
- Originally I tried to just box my services manually. Yeah, don't do this. `tower` already has `BoxCloneService` and `BoxCloneSyncService` for that. You're just going to spend hours reinventing the wheel.
