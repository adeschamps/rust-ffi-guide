# Generating a Header File

Instead of having to constantly keep `ffi.rs` and the various `extern` blocks 
scattered through out our C++ code in sync, it'd be really nice if we could 
generate a header file that corresponds to `ffi.rs` and just `#include` that.
Fortunately there exists a tool which does exactly this called [cbindgen]!


## Adding Cbindgen

You can use `cbindgen` to generate header files in a couple ways, the first is
to use `cargo install` and run the binding generator program.

```
$ cargo install cbindgen
$ cd /path/to/my/project && cbindgen . -o target/my_project.h
```

However running this after every change can get quite repetitive, therefore the
*README* includes a minimal build script which will automatically generate the 
header every time you compile.

First add `cbindgen` as a build dependency ([cargo-edit] makes this quite easy).

```
$ cargo add --build cbindgen
```

You also need to make sure you have a `build` script entry in your `Cargo.toml`.

```diff
...
description = "The business logic for a REST client"
name = "client"
repository = "https://github.com/Michael-F-Bryan/rust-ffi-guide"
version = "0.1.0"
+ build = "build.rs"
+
+ [build-dependencies]
+ cbindgen = "0.1.29"

[dependencies]
chrono = "0.4.0"
...
```

Finally you can flesh out the build script itself. This is fairly 
straightforward, although because we want to put the generated header file in 
the `target/` directory we need to take special care to detect when `cmake` 
overrides the default.

```rust
// client/build.rs

extern crate cbindgen;

use std::env;
use std::path::PathBuf;
use cbindgen::Config;


fn main() {
    let crate_dir = env::var("CARGO_MANIFEST_DIR").unwrap();

    let package_name = env::var("CARGO_PKG_NAME").unwrap();
    let output_file = target_dir()
        .join(format!("{}.hpp", package_name))
        .display()
        .to_string();

    let config = Config {
        namespace: Some(String::from("ffi")),
        ..Default::default()
    };

    cbindgen::generate_with_config(&crate_dir, config)
      .unwrap()
      .write_to_file(&output_file);
}

/// Find the location of the `target/` directory. Note that this may be 
/// overridden by `cmake`, so we also need to check the `CARGO_TARGET_DIR` 
/// variable.
fn target_dir() -> PathBuf {
    if let Ok(target) = env::var("CARGO_TARGET_DIR") {
        PathBuf::from(target)
    } else {
        PathBuf::from(env::var("CARGO_MANIFEST_DIR").unwrap()).join("target")
    }
}
```

Note that the `build.rs` build script also creates a custom `Config` that 
specifies everything in the generated header file should be under the `ffi` 
namespace. This means we won't get name clashes between the opaque `Request` and
`Response` types generated by `cbindgen` and our own wrapper classes.

If you go back to the `build/` directory and recompile, you should now see the 
`client.hpp` header file.

```
$ cd build/
$ cmake -DCMAKE_BUILD_TYPE=Debug ..
$ make
$ ls client
client.hpp  cmake_install.cmake  CMakeFiles  CTestTestfile.cmake  debug  libclient.so  Makefile

$ cat client/client.hpp
#include <cstdint>
#include <cstdlib>

extern "C" {

namespace ffi {

// A HTTP request.
struct Request;

struct Response;

// Initialize the global logger and log to `rest_client.log`.
//
// Note that this is an idempotent function, so you can call it as many
// times as you want and logging will only be initialized the first time.
void initialize_logging();

// Construct a new `Request` which will target the provided URL and fill out
// all other fields with their defaults.
//
// # Note
...
```

This header file is also `#include`-able from all your C++ code, meaning you no
longer need to write all those manual `extern "C"` declarations. It also lets 
you rely on the compiler to do proper type checking instead of getting ugly 
linker errors if your forward declarations become out of sync (or crashes and
data corruption if function arguments change).

To actually `#include` the generated header file we need to make a couple 
adjustments to the `CMakeLists.txt` file to let `cmake` know to add the 
`build/client/` output directory to the include path.


```diff
# gui/CMakeLists.txt

set(CMAKE_INCLUDE_CURRENT_DIR ON)
find_package(Qt5Widgets)

+ set(CLIENT_BUILD_DIR ${CMAKE_BINARY_DIR}/client)
+ include_directories(${CLIENT_BUILD_DIR})
+
set(SOURCE main_window.cpp main_window.hpp wrappers.cpp wrappers.hpp main.cpp)

add_executable(gui ${SOURCE})
```

Now you just need to update the `wrappers.cpp` and `wrappers.hpp` files to 
`#include` this new `client.hpp`, delete the `extern "C"` block, and update the
Rust function call sites to be prefixed with the `ffi::` namespace. As a bonus
we can also replace a bunch of `void *` pointers with proper strongly-typed 
pointers.

This step may take a couple iterations to make sure all the types match up and 
everything compiles again. Make sure to test it all works by running the `gui`
program and hitting our GUI's dummy button. If everything is okay then you 
should see HTML for the Rust website printed to the console.


[cbindgen]: https://github.com/eqrion/cbindgen
[cargo-edit]: https://crates.io/crates/cargo-edit