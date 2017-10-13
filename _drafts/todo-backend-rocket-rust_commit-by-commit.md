---
title: 'Creating a Todo-Backend API with Rocket, Commit-by-Commit'
---

When [Rocket 0.3.0 was announced](https://rocket.rs/news/2017-07-14-version-0.3/),
I was impressed, so when I got some time, I decided to try it out.
The [Todo-Backend Project](http://www.todobackend.com/) provides JavaScript tests for a simple, RESTful, JSON web service
and can be pointed at any implementation,
so it's a good fit for trying out a web framework.
With the [reference specs](http://www.todobackend.com/specs/index.html?http://localhost:8000) loaded in one tab
and the [Rocket Guide](https://rocket.rs/guide/) in another, I got started.

I started out [following the guide](https://rocket.rs/guide/getting-started/#hello-world) exactly.
`cargo new todo-backend-rocket-rust --bin` gave me a [Hello, World! command-line application](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/bba8ba6f1706efc01ea514af4683df893ceafd5c).
I [added rocket and rocket_codegen 0.3.0 as dependencies](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/f1c4d1725e2ea93963f017bf2664d77cfec0a6d2)
and turned command-line Hello, World! [into Rocket Hello, World](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/c52353e030549b4123f4d83d1284072d730a03b8)!

I knew the reference specs wouldn't get far, but the first spec (under "pre-requesites") only checks that
"the api root responds to a GET (i.e. the server is up and accessible, CORS headers are set up),"
so it seemed worth a run.
The parenthesized part of that test name suggested I had some complexity to deal with before I could get to the heart of todo-listing.
Indeed, after refreshing the reference specs tab, they failed hard.

```
AssertionError: expected promise to be fulfilled but it was rejected with [Error: 

GET http://localhost:8000
FAILED

The browser failed entirely when make an AJAX request.
Either there is a network issue in reaching the url, or the
server isn't doing the CORS things it needs to do.
Ensure that you're sending back: 
  - an `access-control-allow-origin: *` header for all requests
  - an `access-control-allow-headers` header which lists headers such as "Content-Type"

Also ensure you are able to respond to OPTION requests appropriately. 

]
```


CORS
---

How do you [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) with Rocket?
After a bit of Googling and reading [this Rocket issue](https://github.com/SergioBenitez/Rocket/issues/25)
(particularly [this comment](https://github.com/SergioBenitez/Rocket/issues/25#issuecomment-316021913))
linking to [rocket_cors](https://github.com/lawliet89/rocket_cors),
I [added a dependency on rocket_cors 0.1.4](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/1539c4f5cae7894ebe3ba62942c2ee98104aadf7)
and [added it to the application](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/f8ac5eca7d0c1a72890f69a602ce9d3b9ead1537)
as a [Fairing](https://rocket.rs/guide/fairings/),
which [the documentation described](https://lawliet89.github.io/rocket_cors/rocket_cors/index.html#fairing) as the simplest, least flexible way to use rocket_cors.

I had no idea if that would be enough to satisfy the first spec, but it turned out to be:
with a refresh, the first spec passed and the second, "the api root responds to a POST with the todo which was posted to it,"
failed with a 404, which made perfect sense, since I had defined only a `GET /` route and it wanted to `POST /`.
I didn't know if that would be CORS enough for the rest of the suite,
but I wanted to explore Rocket's idioms, not CORS, so I moved on.


JSON
---

Next up I wanted to accept and return JSON.
The [Guide's JSON section](https://rocket.rs/guide/requests/#json) led me to the
[official JSON example](https://github.com/SergioBenitez/Rocket/tree/v0.3.0/examples/json),
which showed I needed more dependencies:
[serde, serde_json, and serde_derive 1.0 and rocket_contrib with only the "json" feature](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/625904f4dea9ca5f4e650637332436d5a9af6c17).

With those in place, it was a small step to [get that next spec passing](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/4a0c67bc5dc1a00b031f08c0b43b0f9b3cc98bd0).
I defined a Todo struct that could serialize and deserialize with serde and added a `POST /` endpoint.

```
#[derive(Serialize, Deserialize)]
struct Todo {
    title: String,
}

#[post("/", data = "<todo>")]
fn create_todo(todo: Json<Todo>) -> Json<Todo> {
    todo
}
```

I also discovered at this point that Rocket's compiler plugin provides a lot of help.
Initially I forgot to add my new function to the `mount("/", routes![...])` call in my main method,
and I got this warning.

```
warning: the 'create_todo' route is not mounted
  --> src/main.rs:22:1
   |
22 | / fn create_todo(todo: Json<Todo>) -> Json<Todo> {
23 | |     todo
24 | | }
   | |_^
   |
   = note: #[warn(unmounted_route)] on by default
   = note: Rocket will not dispatch requests to unmounted routes.
help: maybe add a call to `mount` here?
  --> src/main.rs:27:5
   |
27 |     rocket::ignite()
   |     ^^^^^^^^^^^^^^^^
```

That's incredibly helpful!

You'll notice the endpoint wasn't actually doing anything with the `Todo` yet, just echoing back the same `Json` from the request.
That was all the second spec checked for, and the third spec wanted a `DELETE /` route to clear out the list.
That was also easy to [satisfy with a stub implementation](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/6cfce3fa64c49f0fc7fbed4b6e9e176ffa448140).

```
#[delete("/")]
fn delete_all() -> Json<Vec<Todo>> {
    Json(vec![])
}
```

It seemed a bit funny to have the "delete everything" route respond with data,
since even once implemented it should only ever respond with literally `[]`,
but for some reason I thought that's what the specs wanted.
I realized later that the specs just wanted a 200 response code, so `delete_all` didn't have to return anything.

```
#[delete("/")]
fn delete_all() {}
```

For a route defined like that, Rocket responds 200 with `Content-length: 0`.

A similar [stub implementation of `GET /`](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/90be197ea860e082ba518021f30fda7cb2c5b4ee)
got the last of "the pre-requisites" passing.


State
---

The next spec finally required state to carry over from one request to the next.
It did a `POST /` with a todo having only a `title`, then a `GET /` and expected to find a single element array containing an object with that `title`.
I decided to start with an in-memory implementation so I could stay focused on Rocket
and not get side-tracked into persistence (which isn't _yet_ part of Rocket's offering,
though their guide walks through a [good Diesel setup](https://rocket.rs/guide/state/#databases)).

Rocket provides [server-wide state](https://rocket.rs/guide/state/) via its
[`manage` method](https://api.rocket.rs/rocket/struct.Rocket.html#method.manage)
and [`rocket::State` request-guard](https://api.rocket.rs/rocket/struct.State.html).

[Request guards](https://rocket.rs/guide/requests/#request-guards) are an important Rocket concept.
Any type that implements [`rocket::request::FromRequest`](https://api.rocket.rs/rocket/request/trait.FromRequest.html)
can be used as a request guard on a route by adding an argument of that type to the route's handler function. 
As the name suggests, a primary use-case of request guards is to guard against a route being called when it shouldn't be.
The `FromRequest` implementation returns a [`rocket::request::Outcome`](https://api.rocket.rs/rocket/request/type.Outcome.html)
that can indicate either

* that the request can be handled, returning an instance of the implementing type,
* that the request should be rejected, returning an HTTP status and (maybe) an additional error value, or
* that the request can't be handled by this route, but allowing route-matching to continue looking for a route that can handle it.

`State`'s implementation of `FromRequest`, however, doesn't guard anything.
It always provides the managed instance of the requested type, providing a sort of dependency injection.

So, I [gave rocket](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/3f6e1adfb4cdfe1e30dd2ee0bba3addd4b0f8db9#4b569f42a6967dec04275af54f4ca9ab6a4eee64_35_52)
a `Mutex<Vec<Todo>>` to `manage` [and added](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/3f6e1adfb4cdfe1e30dd2ee0bba3addd4b0f8db9#4b569f42a6967dec04275af54f4ca9ab6a4eee64_14_16)
a `State<Mutex<Vec<Todo>>` to the arguments of my `index`, `create_todo`, and `delete_all` handler functions.
(The next spec only used `index` and `create_todo`, but if I didn't also implement `delete_all`,
then it would pass once after a server restart and then fail because there were too many todos in the list.)

`State` implements `Deref` for its wrapped type, so I called [`lock`](https://doc.rust-lang.org/std/sync/struct.Mutex.html#method.lock)
on it directly to be able to read and write the `Vec` of `Todo`s.

Without knowing what was the "right" approach, I cloned the managed state that I needed serialized into responses
so that I could return owned data, not references, and avoid worrying about lifetimes.

Since locking a `Mutex` could fail, I also changed the functions to [return `Result`](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/3f6e1adfb4cdfe1e30dd2ee0bba3addd4b0f8db9#4b569f42a6967dec04275af54f4ca9ab6a4eee64_14_16).
I mapped `Mutex`'s failure to a `rocket::response::Failure` with `Status::InternalServerError`,
but I later realized any `Result::Err` that implements `Debug` gets the same behavior by default.

That got "adds a new todo to the list of todos at the root url" to go green.

Passing the next spec was just a matter of [adding a `completed` property](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/264e52317d37315fdef806607f0fe1e41a7eab9f)
that defaulted to false. Easy.
At this point it started to get awkward to use the same struct for creation requests and API responses.
Every persisted todo should have `completed:true` or `completed:false`, but I made the field an `Option<bool>` because
it wouldn't be included in requests.
(In fact it probably shouldn't be allowed in requests.)

The next spec required another response-only property, `url`,
and I followed the same approach of making it optional in the struct so the struct could be shared between requests and responses.
I held my nose and [generated these URLs in a hacky way](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/3f1d4c43225aac2019df63a74a7e769b1c0d889b).

Next up was [getting single todos via their `url`](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/98f035698c83155db029c997c076bd6ac267b5dc),
which was quick and got the next two specs passing.

The next few specs were about updating a todo with a `PATCH` to its `url`.
If I wanted to use the same `Todo` struct that was already doing double-duty for create requests and API responses,
I'd have to make `title` optional, which was a bridge too far, so I
[created a new `TodoUpdate` struct](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/fe75b2bf9fe675cd56ec8dea99480c437f6fcc4f)
for update requests,
thinking that I'd probably want to do the same for creates once I had all the specs passing.
It started with just a `title`, then I [added `completed`](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/3bd7d26f0e57b754de6b7e773fbd2199beb8b8b8).

The next spec changed gears and only required [adding a `DELETE` endpoint](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/d787622112c862b757ce5f216f298bda6796c442),
which retains all but the todo with the matching url.

Almost there! The next section of specs was about "tracking todo order,"
but surprisingly didn't require actually returning todos in the selected order.
I added an `order` field [to the `Todo` struct](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/f987b201ddc6b27cf983329cbe9151cb9772b7f9)
to handle create and [to the `TodoUpdate` struct, along with some handling code](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/acf71ee9594086bca882e722223c08d3293ca188),
to handle update, and with that, the test suite was all green!


Cleanup
---

Now I finally felt comfortable to start turning this into code I'd actually want to own.

First up, a tiny thing:
I hate a boolean with three states, so I used serde's defaulting functionality to
[default `complete` to false](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/68648312340c84eaeabf14b28b89c9ba18a9941a),
so it doesn't have to be `Option<bool>`.

Then [reordering top-level items](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/c99b3b6070be36f33e39a9cf01a6d57f0de2f06a)
to make this program easier to read.
Several years of working with Clojure left me with the habit of putting `main` functions at
the bottom of a file, but, in languages that support it,
I like to follow the "newspaper metaphor" advice from 'Clean Code'.
(Nicely summarized in [this review](http://www.markhneedham.com/blog/2008/09/15/clean-code-book-review/).)

Next up was the big refactoring I'd been anticipating all along.
Up to now, the application was in a single file, and my web handler functions were littered
with repetitive, hard-to-read boilerplate related to holding data in-memory. 
Instead of each web handler taking a `State<Mutex<Vec<Todo>>>`
and dealing with that unwieldy beast,
I wanted something with a clean API injected into every web handler,
so web code could be completely agnostic about how data was stored.

In one [rather huge commit](https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/83ec235ddbcbaf71f68761bf226d10dcc88f5452), I made several moves to bring this about:

* I created a `TodoList` struct in a new `todo_list` module
  with a single `Mutex<Vec<Todo>>>` field
  and copied most of the guts of every web handler function into methods of the struct.
* I gave rocket a `TodoList` to manage as state instead of a `Mutex<Vec<Todo>>`.
* I implemented `FromRequest` for `TodoList`, pulling it from state [per the guide](https://rocket.rs/guide/state/#within-guards).
* I trimmed down the web handler methods to delegate to `TodoList` and wrap results in `Json` structs for presentation.
* I also chose to introduce a custom error type for the application at this point. (Recall that I had my web handlers mapping a `Mutex` failure to a rocket `Failure`, but I didn't want the new `todo_list` module to know about Rocket. This was when I learned that Rocket will turn any old `Error` into a 500 error, so I could return `Result`s with my custom error type from web handlers, and it would Just Work.)

I was pretty happy with this state. The web handlers were small and easy to understand,
and all knowledge of Rocket was contained in their module, with the `todo_list` module
not depending on Rocket at all. It did have knowledge of JSON serialization with serde, and, worse, generating `Todo` URLs, w/ a hard-coded `localhost` base.

Before tackling the URL thing, I wanted to deal with the lower-hanging fruit of the `Todo`
struct being messy due to doing double-duty representing a request to create a todo and an
already saved todo. I introduced a `TodoCreate` struct to represent the former, so `Todo`
no longer needed to be deserializable, default its `complete` field, or have `url` be optional.



https://gitlab.com/duelinmarkers/todo-backend-rocket-rust/commit/9cc1b2a0f784d3e07c8b9e7c5ffe627d46df4c47
