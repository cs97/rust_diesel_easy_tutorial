# rust_diesel_easy_tutorial
```
cargo new --lib diesel_demo
```

Cargo.toml 
```
[dependencies]
diesel = { version = "2.2.0", features = ["postgres"] }
```



```
podman run --rm --name diesel -p 5432:5432 -e POSTGRES_PASSWORD=postgres -d postgres
```


```
cargo install diesel_cli --no-default-features --features postgres

```




```
~/.cargo/bin/diesel setup --database-url postgres://postgres:postgres@localhost/diesel_demo
```

```
~/.cargo/bin/diesel migration generate create_posts
```


migrations/.../up.sql
```
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  title VARCHAR NOT NULL,
  body TEXT NOT NULL,
  published BOOLEAN NOT NULL DEFAULT FALSE
)
```
migrations/.../down.sql
```
DROP TABLE posts
```

```
~/.cargo/bin/diesel migration run --database-url postgres://postgres:postgres@localhost/diesel_demo
```



src/lib.rs
```
use diesel::prelude::*;
pub mod models;
pub mod schema;

const DATABASE_URL : &str = "postgres://postgres:postgres@localhost/diesel_demo";

pub fn establish_connection() -> PgConnection {
    //dotenv().ok();

    //let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    PgConnection::establish(DATABASE_URL)
        .unwrap_or_else(|_| panic!("Error connecting to {}", DATABASE_URL))
}
```


src/models.rs
```
use diesel::prelude::*;

#[derive(Queryable, Selectable)]
#[diesel(table_name = crate::schema::posts)]
#[diesel(check_for_backend(diesel::pg::Pg))]
pub struct Post {
    pub id: i32,
    pub title: String,
    pub body: String,
    pub published: bool,
}
```

src/bin/show_posts.rs
```
use self::models::*;
use diesel::prelude::*;
use diesel_demo::*;

fn main() {
    use self::schema::posts::dsl::*;

    let connection = &mut establish_connection();
    let results = posts
        .filter(published.eq(true))
        .limit(5)
        .select(Post::as_select())
        .load(connection)
        .expect("Error loading posts");

    println!("Displaying {} posts", results.len());
    for post in results {
        println!("{}", post.title);
        println!("-----------\n");
        println!("{}", post.body);
    }
}
```

src/models.rs
```
use crate::schema::posts;

#[derive(Insertable)]
#[diesel(table_name = posts)]
pub struct NewPost<'a> {
    pub title: &'a str,
    pub body: &'a str,
}
```


src/lib.rs 
```
use self::models::{NewPost, Post};

pub fn create_post(conn: &mut PgConnection, title: &str, body: &str) -> Post {
    use crate::schema::posts;

    let new_post = NewPost { title, body };

    diesel::insert_into(posts::table)
        .values(&new_post)
        .returning(Post::as_returning())
        .get_result(conn)
        .expect("Error saving new post")
}
```



src/bin/write_post.rs
```
use diesel_demo::*;
use std::io::{stdin, Read};

fn main() {
    let connection = &mut establish_connection();

    let mut title = String::new();
    let mut body = String::new();

    println!("What would you like your title to be?");
    stdin().read_line(&mut title).unwrap();
    let title = title.trim_end(); // Remove the trailing newline

    println!("\nOk! Let's write {title} (Press {EOF} when finished)\n",);
    stdin().read_to_string(&mut body).unwrap();

    let post = create_post(connection, title, &body);
    println!("\nSaved draft {title} with id {}", post.id);
}

#[cfg(not(windows))]
const EOF: &str = "CTRL+D";

#[cfg(windows)]
const EOF: &str = "CTRL+Z";
```






src/bin/publish_post.rs
```
use self::models::Post;
use diesel::prelude::*;
use diesel_demo::*;
use std::env::args;

fn main() {
    use self::schema::posts::dsl::{posts, published};

    let id = args()
        .nth(1)
        .expect("publish_post requires a post id")
        .parse::<i32>()
        .expect("Invalid ID");
    let connection = &mut establish_connection();

    let post = diesel::update(posts.find(id))
        .set(published.eq(true))
        .returning(Post::as_returning())
        .get_result(connection)
        .unwrap();
    println!("Published post {}", post.title);
}
```


src/bin/get_post.rs
```
use self::models::Post;
use diesel::prelude::*;
use diesel_demo::*;
use std::env::args;

fn main() {
    use self::schema::posts::dsl::posts;

    let post_id = args()
        .nth(1)
        .expect("get_post requires a post id")
        .parse::<i32>()
        .expect("Invalid ID");

    let connection = &mut establish_connection();

    let post = posts
        .find(post_id)
        .select(Post::as_select())
        .first(connection)
        .optional(); // This allows for returning an Option<Post>, otherwise it will throw an error

    match post {
        Ok(Some(post)) => println!("Post with id: {} has a title: {}", post.id, post.title),
        Ok(None) => println!("Unable to find post {}", post_id),
        Err(_) => println!("An error occured while fetching post {}", post_id),
    }
}
```


 src/bin/delete_post.rs
```
use diesel::prelude::*;
use diesel_demo::*;
use std::env::args;

fn main() {
    use self::schema::posts::dsl::*;

    let target = args().nth(1).expect("Expected a target to match against");
    let pattern = format!("%{}%", target);

    let connection = &mut establish_connection();
    let num_deleted = diesel::delete(posts.filter(title.like(pattern)))
        .execute(connection)
        .expect("Error deleting posts");

    println!("Deleted {} posts", num_deleted);
}
```










