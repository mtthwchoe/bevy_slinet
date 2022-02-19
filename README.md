# bevy_slinet
A simple networking plugin for bevy.

[![docs.rs](https://img.shields.io/docsrs/bevy_slinet)](https://docs.rs/bevy_slinet)
[![Crates.io](https://img.shields.io/crates/v/bevy_slinet)](https://crates.io/crates/bevy_slinet)
[![Crates.io](https://img.shields.io/crates/l/bevy_slinet)](https://github.com/Sliman4/bevy_slinet/tree/main/LICENSE)

## Features
- You can choose TCP or UDP protocol, with websocket (or other wasm-friendly solution?) planned. Adding your own protocols is as easy as implementing a few traits.
- Multiple clients/servers with different configs (config is a collection of a protocol, packet types, a serializer, etc.)
- You don't have to handle deserialization manually: you choose a protocol, serializer, packet type, and we do everything for you!


## Client example
```rust
use bevy::app::{App, EventReader};
use bevy::MinimalPlugins;
use bincode::DefaultOptions;
use serde::{Deserialize, Serialize};

use bevy_slinet::client::{ClientPlugin, PacketReceiveEvent};
use bevy_slinet::packet_length_serializer::LittleEndian;
use bevy_slinet::protocols::tcp::TcpProtocol;
use bevy_slinet::serializers::bincode::BincodeSerializer;
use bevy_slinet::ClientConfig;

struct Config;

impl ClientConfig for Config {
    type ClientPacket = ClientPacket;
    type ServerPacket = ServerPacket;
    type Protocol = TcpProtocol;
    type Serializer = BincodeSerializer<DefaultOptions>;
    type LengthSerializer = LittleEndian<u32>;
}

#[derive(Serialize, Deserialize, Debug)]
enum ClientPacket {
    String(String),
}

#[derive(Serialize, Deserialize, Debug)]
enum ServerPacket {
    String(String),
}

fn main() {
    App::new()
        .add_plugins(MinimalPlugins)
        .add_plugin(ClientPlugin::<Config>::connect("127.0.0.1:3000"))
        .add_system(connection_establish_system)
        .add_system(packet_receive_system)
        .run()
}

fn connection_establish_system(mut events: EventReader<ConnectionEstablishEvent<Config>>) {
    for event in events.iter() {
        println!("Connected!");
    }
}

fn packet_receive_system(mut events: EventReader<PacketReceiveEvent<Config>>) {
    for event in events.iter() {
        match &event.packet {
            ServerPacket::String(s) => println!("Got a message: {}", s),
        }
        event
            .connection
            .send(ClientPacket::String("Hello, Server!".to_string()));
    }
}
```

## Server Example

```rust
use bevy::app::{App, EventReader};
use bevy::MinimalPlugins;
use bincode::DefaultOptions;
use serde::{Deserialize, Serialize};

use bevy_slinet::packet_length_serializer::LittleEndian;
use bevy_slinet::protocols::tcp::TcpProtocol;
use bevy_slinet::serializers::bincode::BincodeSerializer;
use bevy_slinet::server::{NewConnectionEvent, ServerPlugin, PacketReceiveEvent};
use bevy_slinet::ServerConfig;

struct Config;

impl ServerConfig for Config {
    type ClientPacket = ClientPacket;
    type ServerPacket = ServerPacket;
    type Protocol = TcpProtocol;
    type Serializer = BincodeSerializer<DefaultOptions>;
    type LengthSerializer = LittleEndian<u32>;
}

#[derive(Serialize, Deserialize, Debug)]
enum ClientPacket {
    String(String),
}

#[derive(Serialize, Deserialize, Debug)]
enum ServerPacket {
    String(String),
}

fn main() {
    App::new()
        .add_plugins(MinimalPlugins)
        .add_plugin(ServerPlugin::<Config>::bind("127.0.0.1:3000").unwrap())
        .add_system(new_connection_system)
        .add_system(packet_receive_system)
        .run()
}

fn new_connection_system(mut events: EventReader<NewConnectionEvent<Config>>) {
    for event in events.iter() {
        event
            .connection
            .send(ServerPacket::String("Hello, Client!".to_string()));
    }
}

fn packet_receive_system(mut events: EventReader<PacketReceiveEvent<Config>>) {
    for event in events.iter() {
        match &event.packet {
            ClientPacket::String(s) => println!("Got a message from a client: {}", s),
        }
        event
            .connection
            .send(ServerPacket::String("Hello, Client!".to_string()));
    }
}
```

Note: you should implement keep-alive and disconnection systems yourself, or look at [lobby_and_battle_servers example](examples/lobby_and_battle_servers.rs)

## More examples
[Here](https://github.com/Sliman4/bevy_slinet/tree/main/examples).

### Compatibility table
| Plugin Version | Bevy Version |
|----------------|--------------|
| `0.1`          | `0.6`        |
| `main`         | `main`       |