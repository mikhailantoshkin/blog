+++
title="shnek?"
date=2024-12-14
draft=false

[extra]
disable_comments = true
+++

# Setup
For initial setup we'll need a few tools. First is I highly recommend [uv](https://docs.astral.sh/uv/) to manage python and venv. Long story short it's `cargo for python` build by `astral`.
Next we'll need a build tool for python extension modules build with Rust. I chose [maturin](https://github.com/PyO3/maturin) as most straightforward one.

First, lets setup our project. uv makes its very easy, just run
```
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
The idea is simple. Root of the project contains all the "project related" files, like README, Dockerfiles, CI scripts and whatever else. All the code is contained in the directories named after the modules.
Directory `shnek` is containing all our source code and consists of:

`__main__.py` that allows to run the whole directory as a module with `python -m shnek` and looks like this
```
from shnek.main import main

if __name__ == '__main__':
    main()
```
`main.py` will be holding our entrypoint and looks like this
```
def main():
    print('Hello shnek')
```
Python is not that strict in terms of project structure, so you can use whatever structure you prefer. I just happen to prefer this over `src` pattern. 

Now running the  `python -m shnek` in the root of the project produces `Hello shnek`. Good start, let's move on to setting up our rust library

## Setup of rust lib
Maturin allows multiple ways to setup mixed Python/Rust project. For my purposes I found that creating pure rust sub-module works the best. Since this package is not going to be published on pypi it doesn't really matter, but
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
Now running `maturin develop --uv` from `shnek_lib` will compile the module and install it into local venv, allowing you to import it in Python code. The only problem that extension modules don't actually carry any
type and function signature information with them. So you can import them and it will actually run, but you going to lose autocomplete and typecheking. To fix this we need to provide "type stubs" for our extension module. 
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
Neet, it works. Let's make it do something a bit more useful.



