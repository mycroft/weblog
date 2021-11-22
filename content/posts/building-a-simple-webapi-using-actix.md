---
title: "Building a Simple Webapi Using Actix"
date: 2021-11-22T11:32:13+01:00
summary: "Building a real simple API using Actix"
categories:
  - rust
---

In this short tutorial, I'll give some bits to build a small but complete web API using Rust & Actix Web. My goal with this was to build a sample to-do application from scratch, and scratch the surface of async, web related crates & sql related ones.

I'll be most likely using official documentation (though it is kinda outdated and would need a serious refresh in my opinion), which means serde, r2d2, pretty_env_logger, failure crates will be used in this.


# Deploy the tutorial app

Firstly, we begin with starting a brand new project & tries the first tutorial sample. I'll name the project `augias`, related to this king in the greek mythology for his infamous stables. My to-do list is most likely in the same shape of the stables.

```sh
$ cargo new augias && cd augias
    Created binary (application) `augias` package
```

After adding `actix-web` to `Cargo.toml`, file will look like:

```toml
[dependencies]
actix-web = "3"
```

And we'll use at first the "Get Started" sample code. I've added some comments inline.

```rust
use actix_web::{web, App, HttpRequest, HttpServer, Responder};

// An handler is async and can return any object that implements the Responder trait. 
async fn greet(req: HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    format!("Hello {}!", &name)
}

// actix is async.
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            // Add a new route for "get" method handled by the "greet" handler.
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
    })
    // bind localhost/8080 ports
    .bind(("127.0.0.1", 8080))?
    // run & await
    .run()
    .await
}
```

See [Responder trait](https://docs.rs/actix-web/3.3.2/actix_web/trait.Responder.html)


# Preparing & using the database backend

The API will do 2 things (for now): Insert tasks & retrieve them. It is so required to define 2 functions to handle the database needs: One for inserting, one for retrieving them all.

Database connections will use [r2d2](https://docs.rs/crate/r2d2) crate. I'll be using `r2d2_sqlite` & [failure](https://docs.rs/failure) for making a bit easier error management, and start by creating the database:

```sh
$ sqlite3 tasks.db
sqlite> create table tasks(name varchar(32) PRIMARY KEY, content text);
sqlite> insert into tasks(name, content) values('first task', 'this is the content of my first task');
sqlite> select * from tasks;
first task|this is the content of my first task
sqlite>
```

Then we add a first function to retrieve the tasks:

```rust
use failure::Error;

use r2d2::PooledConnection;
use r2d2_sqlite::SqliteConnectionManager;

type DbPool = PooledConnection<SqliteConnectionManager>;

pub fn get_all_tasks(conn: &DbPool) -> Result<Vec<String>, Error> {
    let mut stmt = conn.prepare("SELECT name FROM tasks")?;

    let tasks = stmt.query_map([], |row| {
        row.get(0)
    })?;

    Ok(tasks.into_iter().map(Result::unwrap).collect())
}
```

Function `get_all_tasks` is either returning an Error or fetches all tasks names into an array. Once this done, we need to adapt the main module to handle database connection:

```rust
use r2d2::Pool;
use r2d2_sqlite::SqliteConnectionManager;

mod db;

// list is a new web handler. It includes the data structure passed thanks to the .data call
// in the App::new() statement in the main function.
async fn list(data: web::Data<Pool<SqliteConnectionManager>>, _req: HttpRequest) -> impl Responder {
    let conn = data
        .get()
        .expect("couldn't get db connection from pool");

    // It is required to use web::block here to make sure the database call, which is not an async call,
    // is not done in the general context
    let pages_res = web::block(move || db::get_all_tasks(&conn))
        .await;

    // Returns the result in json format
    match pages_res {
        Ok(pages) => HttpResponse::Ok().json(pages),
        Err(_err) => {
            HttpResponse::InternalServerError().finish()
        }
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Prepare the database access connections pool
    let db_manager = SqliteConnectionManager::file("tasks.db");

    let db_pool = Pool::builder()
        .build(db_manager)
        .expect("Failed to create pool.");

    // "move" was added here to pass the db_pool ownership
    HttpServer::new(move || {
        App::new()
            // cloning the pool here, as it doesn't implement copy.
            .data(db_pool.clone())
            // updating the handler to target our new "list" handler
            .route("/", web::get().to(list))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

What changed here:
- We added required crate usage for `r2d2` & `r2d2_sqlite`, and a reference the the db module that contains our database functions;
- We created a web handler that would retrieve the tasks;
- We created a db connection handler pool & passing it to the web app.

After a build & a run, the first iteration of our web application is now correctly running. We can verify that by using HTTPie:

```sh
$ http "http://localhost:8080/"
HTTP/1.1 200 OK
content-length: 28
content-type: application/json
date: Mon, 22 Nov 2021 11:37:22 GMT

[
    "first task",
]
```


# Adding tasks

Next is to create a way of adding new tasks. First, we add `pretty_env_logger` & `serde_derive` crates as we now want better log messages & will need a way to parse inputs. The `Cargo.toml` now looks like:

```toml
[dependencies]
actix-web = "3"
failure = "0.1.8"
log = "0.4.14"
pretty_env_logger = "0.4.0"
r2d2 = "0.8.9"
r2d2_sqlite = "0.19.0"
serde = "1.0.130"
serde_derive = "1.0.130"
```

Our `db.rs` module get expanded with a new `add_task` function:

```rust
type DbPool = PooledConnection<SqliteConnectionManager>;

pub fn add_task(conn: &DbPool, name: &str, content: &str) -> Result<bool, Error> {
    match conn.execute(
        "INSERT INTO tasks(name, content) VALUES (?, ?);",
        [name, content],
    ) {
        Ok(_x) => {
            info!("Creating new task '{}'", name);
            Ok(true)
        }
        Err(err) => {
            error!("Error: {:?}", err);
            Err(Error::from(err))
        }
    }
}
```

Note that we're making use of [log](https://docs.rs/log/)'s `info!` & `error!` without defining any crate using it. This is done in the root module of the project, meaning `main.rs`:

```rust
// ... snip ...
use serde_derive::Deserialize;

extern crate pretty_env_logger;
#[macro_use] extern crate log;

// ... snip ...

#[derive(Deserialize)]
struct NewFormData {
    name: String,
    content: String,
}

async fn add(form: web::Form<NewFormData>, data: web::Data<Pool<SqliteConnectionManager>>) -> impl Responder {
    let conn = data
        .get()
        .expect("couldn't get db connection from pool");

    info!("Adding new task {}", form.name);

    let task_res = web::block(move || {
            db::add_task(&conn, &form.name, &form.content)
        })
        .await;

    match task_res {
        Ok(_res) => {
            HttpResponse::Ok().json("Created a new task")
        },
        Err(_res) => {
            HttpResponse::Conflict().json("Could not create new task")
        }
    }
}

// ... snip ...

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    pretty_env_logger::init();

// ... snip ...

            .route("/", web::get().to(list))
            .route("/add", web::post().to(add))
    })

// ... snip ...

```

A log of code was added here! Let's debrief what was added:

* We added the serde_derive & log crates & macros. We need serde_derive to understand the request body when adding tasks (see the `NewFormData` structure).
* The `add` handler was added. Like the former `list` handler, we're retrieve the db connection, then add the task in a `web::block` call, moving in it our form contents, then check the result & return a status.
* Finally, in the `main` function, we initialize the `pretty_env_logger`, then we added the new `/add` `POST` API endpoint.

Trying it:

```sh
$ http --form http://localhost:8080/add "name=second_task" "content=This is another task"
HTTP/1.1 200 OK
content-length: 20
content-type: application/json
date: Mon, 22 Nov 2021 12:14:45 GMT

"Created a new task"
```

In the `cargo run` console, the log messages started to appear as well:

```sh
$ RUST_LOG="warn,augias=trace" cargo run
   Compiling augias v0.1.0 (/home/mycroft/tmp/augias)
    Finished dev [unoptimized + debuginfo] target(s) in 4.02s
     Running `target/debug/augias`
 INFO  augias > Adding new task second_task
 INFO  augias::db > Creating new task 'second_task'
```

If we retry to create a task with the same name, it will fail because of the PRIMARY KEY on the `name` column:

```sh
$ http --form http://localhost:8080/add "name=second_task" "content=This is another task"
HTTP/1.1 409 Conflict
content-length: 27
content-type: application/json
date: Mon, 22 Nov 2021 12:21:38 GMT

"Could not create new task"
```

Again, we got some new error message:

```sh
 INFO  augias     > Adding new task second_task
 ERROR augias::db > Error: SqliteFailure(Error { code: ConstraintViolation, extended_code: 1555 }, Some("UNIQUE constraint failed: tasks.name"))
```


# Deleting a single task

Okay, we can read tasks, we can add some. Last action I wanted to add is to delete a task. Let's prepare our DB function first:

```rust
pub fn delete_task(conn: &DbPool, name: &str) -> Result<bool, Error> {
    match conn.execute(
        "DELETE FROM tasks WHERE name = ?",
        params![name],
    ) {
        Ok(deleted) => {
            info!("Delete execution returned {}", deleted);
            Ok(deleted == 1)
        },
        Err(err) => {
            error!("Delete returned an error: {:?}", err);
            Err(Error::from(err))
        }
    }
}
```

Nothing really new here, as it is pretty similar to the INSERT query above.

Then, let's add our new web handler & route it in the web app:

```rust

// ... snip ...

async fn delete(req: HttpRequest, data: web::Data<Pool<SqliteConnectionManager>>) -> impl Responder {
    // We retrieve the the name from the request url:
    let name = req.match_info().get("name");
    let conn = data
        .get()
        .expect("couldn't get db connection from pool");

    if let None = name {
        return HttpResponse::NotFound().body("Could not find task");
    }

    // We "copy" the name, as the HttpRequest may disappear.
    let deleted_name = name.unwrap().to_string();
    warn!("Deleting task '{}'", deleted_name);

    let task_res = web::block(move || {
            db::delete_task(&conn, &deleted_name)
        })
        .await;

    match task_res {
        Ok(res) => {
            if res {
                HttpResponse::Ok().json("done")
            } else {
                HttpResponse::NotFound().body("could not delete task")
            }
        },
        Err(_res) => HttpResponse::Conflict().json("An error happened"),
    }
}

// ... snip ...

            .route("/add", web::post().to(add))
            // This new route is add a path parameter named name:
            .route("/{name}/delete", web::delete().to(delete))
    })

// ... snip ...
```

And when testing:

```sh
$ http DELETE http://localhost:8080/second_task/delete
HTTP/1.1 200 OK
content-length: 6
content-type: application/json
date: Mon, 22 Nov 2021 12:42:59 GMT

"done"

$ http DELETE http://localhost:8080/second_task/delete
HTTP/1.1 404 Not Found
content-length: 21
date: Mon, 22 Nov 2021 12:43:01 GMT

could not delete task

$ http http://localhost:8080/
HTTP/1.1 200 OK
content-length: 14
content-type: application/json
date: Mon, 22 Nov 2021 12:47:18 GMT

[
    "first task"
]
```

Logs will look like:

```sh
$ RUST_LOG="warn,augias=trace" cargo run
   Compiling augias v0.1.0 (/home/mycroft/tmp/augias)
    Finished dev [unoptimized + debuginfo] target(s) in 4.05s
     Running `target/debug/augias`
 WARN  augias > Deleting task 'second_task'
 INFO  augias::db > Delete execution returned 1
 WARN  augias     > Deleting task 'second_task'
 INFO  augias::db > Delete execution returned 0
 INFO  augias     > Retrieve list of all tasks.
```


# References:

* [Actix web](https://actix.rs)
* [Create a blazingly fast api in Rust](https://hub.qovery.com/guides/tutorial/create-a-blazingly-fast-api-in-rust-part-1/)
