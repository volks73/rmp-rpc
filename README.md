rmp-rpc
=======

A Rust implementation of MessagePack-RPC inspired by
[`msgpack-rpc-rust`](https://github.com/euclio/msgpack-rpc-rust), but based on
[tokio](http://tokio.rs/).

[Documentation](https://docs.rs/rmp-rpc/0.0.2/rmp_rpc/index.html).


Example
-------

Here is a minimalistic example showing how to write a simple server and client.
For a more advanced example you can take a look at the
[calculator example](examples/calculator).

```rust
extern crate futures;
extern crate tokio_core;
extern crate tokio_proto;
extern crate rmp_rpc;


use tokio_proto::TcpServer;
use rmp_rpc::{Server, Protocol, Dispatch, Client};
use rmp_rpc::msgpack::{Value, Utf8String};
use tokio_core::reactor::Core;
use std::thread;
use std::time::Duration;
use futures::Future;


#[derive(Clone)]
pub struct HelloWorld;

// a server with two methods, "hello" that returns "hello", and "world" that return "world"
impl Dispatch for HelloWorld {
    fn dispatch(&mut self, method: &str, _params: &[Value]) -> Result<Value, Value> {
        match method {
            "hello" => { Ok(Value::String(Utf8String::from("hello"))) }
            "world" => { Ok(Value::String(Utf8String::from("world"))) }
            _ => { Err(Value::String(Utf8String::from(format!("Invalid method {}", method)))) }
        }
    }
}

fn main() {
    let addr = "127.0.0.1:54321".parse().unwrap();

    // start the server in the background
    thread::spawn(move || {
        let tcp_server = TcpServer::new(Protocol, addr);
        tcp_server.serve(|| {
            Ok(Server::new(HelloWorld))
        });
    });

    thread::sleep(Duration::from_millis(100));

    // client part

    let mut core = Core::new().unwrap();
    let handle = core.handle();
    core.run(
        Client::connect(&addr, &handle)
            .and_then(|client| {
                client.request("hello", vec![])
                    .and_then(move |response| {
                        println!("client: {:?}", response);
                        client.request("world", vec![])
                    })
                    .and_then(|response| {
                        println!("client: {:?}", response);
                        Ok(())
                    })
            })
            ).unwrap();
}
```
