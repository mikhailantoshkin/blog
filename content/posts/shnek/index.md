+++
title="Snake carcinization: Python/Rust hybrid backend with uv and PyO3"
date=2024-12-14
draft=false

[extra]
disable_comments = true
+++

# Foreword

For the longest time I wanted to experiment with building a Python/Rust hybrid backend. This interest
does not come from the idle though: I am somewhat of a Rust advocate. 

But enough of the idle chat, let's get building!

# Setup

For initial setup we'll need a few tools. First, I highly recommend [uv](https://docs.astral.sh/uv/) to manage python and venv.
Long story short it's `cargo for python` build by `astral`. Next we'll need a build tool for python extension modules
written with Rust. I chose [maturin](https://github.com/PyO3/maturin) as most straightforward one.

Lets setup our project. uv makes its very easy, just run

```sh
uv init shnek
cd shnek
uv venv
source .venv/bin/activate
``` 

and you have a skeleton project looking like this

```sh
../shnek
├── hello.py
├── pyproject.toml
├── README.md
└── writup.md
```

Good starting point, but for python backend I prefer a bit different structure that looks like this.

```sh
../shnek
├── pyproject.toml
├── README.md
└── shnek
    ├── __init__.py
    ├── __main__.py
    └── main.py
```

The idea is simple. Root of the project contains all the "project related" files, like README, Dockerfiles, CI scripts and whatever else.
All the code is contained in the directories named after the modules. Directory `shnek` is containing all our source code and consists of:

`__main__.py` that allows to run the whole directory as a module with `python -m shnek` and looks like this

```python
from shnek.main import main

if __name__ == '__main__':
    main()
```

`main.py` will be holding our entrypoint and looks like this

```python
def main():
    print('Hello shnek')
```

Python is not that strict in terms of project structure, so you can use whatever structure you prefer. I just happen to prefer this over `src` pattern. 

Now running the  `python -m shnek` in the root of the project produces `Hello shnek`. Good start, let's move on to setting up our rust library

## Setup of rust lib

Maturin allows multiple ways to setup mixed Python/Rust project. For my purposes I found that creating pure rust
sub-module works the best. Since this package is not going to be published on pypi it doesn't really matter, but
if you building a project intended to be published as a single package - look into options for mixed projects provided by maturin.

`maturin init shnek_lib -b pyo3` will create a `shnek_lib` directory containing Rust project ready to be build into python module.

```sh
../shnek
├── pyproject.toml
├── README.md
├── shnek
│   ├── __init__.py
│   ├── __main__.py
│   └── main.py
└── shnek_lib
    ├── Cargo.toml
    ├── pyproject.toml
    └── src
        └── lib.rs
```

Now running `maturin develop --uv` from `shnek_lib` will compile the module and install it into local venv, allowing you to import it in Python code.
The only problem that extension modules don't actually carry any type and function signature information with them.
So you can import them and it will actually run, but you going to lose autocomplete and typecheking. To fix this we need to provide "type stubs" for our extension module. 
This is done with `.pyi` files containing typedefs and function signatures. So lets create type stub for our module called `shnek_lib.pyi` with following contents

```python
def sum_as_string(a: int, b: int) -> str:
    ...
```

Now when we try to import the function from extension module IDE generates helpful docs. With this part out of the way lets change our `main.py` to run the code from extension module like so

```python
from shnek_lib.shnek_lib import sum_as_string

def main():
    print('Hello shnek')
    print(sum_as_string(1, 2))
```

Now running the  `python -m shnek` in the root of the project produces

```sh
Hello shnek
3
```

Neat, it works. Let's make it do something a bit more useful.


# Doing stuff

Let's make it do something more complex. For this example I chose to build an async syslog server. Why?
Because I've actually had to build one and the last time I've touched it - it already had every drop
of performance squeezed out of it. The next step in it's life cycle would be a rewrite.

Let's start simple: just a server that accepts incoming requests and prints them to stdout. Our `main.py`
will contain the following:

```python
from asyncio import StreamReader, StreamWriter, start_server, run

PORT = 8888


def main():
    run(run_server())


def parse(data: bytes):
    print(data.decode())


async def handler(reader: StreamReader, writer: StreamWriter):
    addr = writer.get_extra_info("peername")
    print(f"Got connection from {addr}")
    while data := await reader.readuntil(b"\n"):
        parse(data[:-1])


async def run_server():
    server = await start_server(handler, "127.0.0.1", PORT)

    print(f"Running server on {PORT}")

    async with server:
        await server.serve_forever()
```

Let's check if it works. Running it with `python -m shnek` and you can send some data to it

```off
Running server on 8888
Got connection from ('127.0.0.1', 49898)
<34>1 2024-12-15T20:57:00.217+09:00 mantoshkin-pc spamlog 12095 0 - hello from rust!
<34>1 2024-12-15T20:57:00.217+09:00 mantoshkin-pc spamlog 12095 1 - hello from rust!
<34>1 2024-12-15T20:57:01.217+09:00 mantoshkin-pc spamlog 12095 2 - hello from rust!
<34>1 2024-12-15T20:57:02.217+09:00 mantoshkin-pc spamlog 12095 3 - hello from rust!
<34>1 2024-12-15T20:57:03.217+09:00 mantoshkin-pc spamlog 12095 4 - hello from rust!
```

Works as expected. Though it does not handle any errors when peer disconnects on partial write, but we are not
writing production code here, so its fine for the experiments.

Now when we have a server lets make it do work.

## Syslog parsing

__intro into syslog and how no-one is following RFC 3164__

So for the sake of the experiment (and my sanity) lets go with RFC 5424. Gladly there is a package available for
parsing syslog 5424 aptly named `syslog-rfc5424-parser`. Let's add it to our project

```sh
uv add syslog-rfc5424-parser
```

And actually start parsing the incoming messages:

```python
from syslog_rfc5424_parser import SyslogMessage, ParseError

def parse(data: bytes):
    msg = SyslogMessage.parse(data.decode())
    print(msg.as_dict())
```
Running a server we get a dictionary representation of the syslog message

```off
Running server on 8888
Got connection from ('127.0.0.1', 47972)
{'severity': 'crit', 'facility': 'auth', 'version': 1, 'timestamp': '2024-12-15T21:08:09.484+09:00', 'hostname': 'mantoshkin-pc', 'appname': 'spamlog', 'procid': 18895, 'msgid': '0', 'sd': {}, 'msg': 'hello from rust!'}
```

## Simulating integration with rest of the system

Syslog server by it self is not very interesting. To reflect real world a little better we'll simulate integration
with the rest of the system by sending the parsed messages to a message broker. 

For the message broker lets use NATS

To work with nats we need a few more dependencies, namely `nats-py` and `orjson`

```
uv add nats-py orjson
```

To make it work though we'll need to refactor our code a bit. Since handler function needs access to an
instance of nats `Client` class and it is returned by the async call to `start_server` we'll have to be a bit clever.
One option is to make everything a class that holds a `Client` as an attribute. The other one is to make handler function a closure so it can capture the `Client`. 
Let's opt in for a class solution, just because it's a bit easier to reason about. After all the refactoring our `main.py` looks something like this

```python
from asyncio import StreamReader, StreamWriter, start_server, run
from syslog_rfc5424_parser import SyslogMessage
import nats
from orjson import dumps

PORT = 8888


def main():
    server = Server()
    run(server.serve())


class Server:
    def __init__(self) -> None:
        self.nc = None
        self.server = None

    async def serve(self):
        self.server = await start_server(self.handler, "127.0.0.1", PORT)
        self.nc = await nats.connect("nats://localhost:4222")

        print(f"Running server on {PORT}")

        async with self.server:
            await self.server.serve_forever()

    async def handler(self, reader: StreamReader, writer: StreamWriter):
        addr = writer.get_extra_info("peername")
        print(f"Got connection from {addr}")
        while data := await reader.readuntil(b"\n"):
            msg = SyslogMessage.parse(data.decode())
            await self.nc.publish("foo", dumps(msg.as_dict()))

```

Now running server and hitting it with some load we can see the message throughput in `nats-top`. Emulation
of integration and benchmarking in one go, neet. Let's take this chance to benchmark this solution. 

```off
NATS server version 2.10.20 (uptime: 27m50s)
Server:
  ID:   NDK5QJWY674R2LJFNOAE7RMGCBDVMK5YXIDWHQA3WLQKGFR6UEPJCND5
  Load: CPU:  0.0%  Memory: 17.7M  Slow Consumers: 0
  In:   Msgs: 1.2M  Bytes: 226.5M  Msgs/Sec: 16390.2  Bytes/Sec: 3.2M
  Out:  Msgs: 0  Bytes: 0  Msgs/Sec: 0.0  Bytes/Sec: 0

Connections Polled: 1
  HOST             CID    SUBS    PENDING     MSGS_TO     MSGS_FROM   BYTES_TO
  ::1:40536        6      0       0           0           1.2M        0
```

With full throttle spamming of messages we get somewhere around 16k msgs/sec. This is going our baseline for later. 

> Don't look to hard into absolute values, it's python, it's running in a single thread,
> so we going to be looking for a rough comparison. Every % of CPU usage won allows other stuff to run in it's place. 

## Moving stuff to rust
Before we start optimizing anything let's first check <> the solution. For this we going to utilize `py-spy`

```sh
uv tool install py-spy
```

```sh
py-spy record -p 42275 -o shnek.svg
```

This gives us pretty flamegraph to look at

![](shnek.svg "Flamegraph")

As expected the bulk of the work is parsing the message. So lets do the parsing on the Rust side!

Luckily for us the package we are using for parsing messages also available in rust as `syslog_rfc5424` and
it has a feature to serialize the syslog messages with `serde`. Let's add it and grab a serializer implementation
while we are at it.

```sh
cargo add syslog_rfc5424 --features serde-serialize
cargo add serde_json
```

Now lets implement the parsing on the rust side

```rust
use pyo3::prelude::*;
use syslog_rfc5424::parse_message;
use serde_json;

#[pyfunction]
fn parse(data: String) -> PyResult<String> {
    let msg = parse_message(data).unwrap();
    return Ok(serde_json::to_string(&msg).expect("Should encode to JSON"));
}

#[pymodule]
fn shnek_lib(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(parse, m)?)?;
    Ok(())
}
```

And not to forget to update the typing information

```python
def parse(data: str) -> str: ...
```

On the python side the changes are minimal

```python
from asyncio import StreamReader, StreamWriter, start_server, run
import nats
from shnek_lib.shnek_lib import parse

PORT = 8888


def main():
    server = Server()
    run(server.serve())


class Server:
    def __init__(self) -> None:
        self.nc = None
        self.server = None

    async def serve(self):
        self.server = await start_server(self.handler, "127.0.0.1", PORT)
        self.nc = await nats.connect("nats://localhost:4222")

        print(f"Running server on {PORT}")

        async with self.server:
            await self.server.serve_forever()

    async def handler(self, reader: StreamReader, writer: StreamWriter):
        addr = writer.get_extra_info("peername")
        print(f"Got connection from {addr}")
        while data := await reader.readuntil(b"\n"):
            msg = parse(data.decode())
            await self.nc.publish("foo", msg.encode())
```

Now let's take a look at the speed

```off
NATS server version 2.10.20 (uptime: 58m58s)
Server:
  ID:   NDK5QJWY674R2LJFNOAE7RMGCBDVMK5YXIDWHQA3WLQKGFR6UEPJCND5
  Load: CPU:  0.0%  Memory: 18.4M  Slow Consumers: 0
  In:   Msgs: 3.0M  Bytes: 593.6M  Msgs/Sec: 65482.9  Bytes/Sec: 13.3M
  Out:  Msgs: 0  Bytes: 0  Msgs/Sec: 0.0  Bytes/Sec: 0

Connections Polled: 1
  HOST             CID    SUBS    PENDING     MSGS_TO     MSGS_FROM   BYTES_TO
  ::1:45846        9      0       0           0           756.3K      0
```

Almost a five times improvement. Not bad for 14 lines of code! For a bunch of projects speed improvement like this
would be plenty enough to keep it afloat for a few release cycles.
