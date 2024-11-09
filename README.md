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

```
~/.cargo/bin/diesel migration run --database-url postgres://postgres:postgres@localhost/diesel_demo
```




```
```


