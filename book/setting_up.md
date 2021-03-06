# Setting Up

Before we can start doing any coding we need to get a build environment set up
and run a hello world program to check everything works.

This chapter will cover:

- Setting up a C++ build system
- Integrating `cargo` into the build system transparently
- A "hello world" to test that C++ can call Rust functions


## Setting up Qt and the Build System

First, create a new `cmake` project in a directory of your choosing.

```bash
$ mkdir rest_client && cd rest_client
$ mkdir gui
$ touch gui/main.cpp
$ touch CMakeLists.txt
```

You'll then want to make sure your `CMakeLists.txt` file (the file specifying 
the project and build settings) looks something like this.

```
# CMakeLists.txt

cmake_minimum_required(VERSION 3.7)
project(rest-client)

enable_testing()
add_subdirectory(client)
add_subdirectory(gui)
```

This says we're building a project called `rest-client` that requires at least 
`cmake` version 3.7. We've also enabled testing and added two subdirectories to
the project (`client` and `gui`).

Our `main.cpp` is still empty, lets rectify that by adding in a [button].


```cpp
// gui/main.cpp

#include <QtWidgets/QPushButton>
#include <QtWidgets/QApplication>

int main(int argc, char **argv) {
  QApplication app(argc, argv);

  QPushButton button("Hello World");
  button.show();

  app.exec();
}
```

We need to add a `CMakeLists.txt` to the `gui/` directory to let `cmake` know
how to build our GUI.

```cmake
# gui/CMakeLists.txt

set(CMAKE_CXX_STANDARD 14)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
find_package(Qt5Widgets)

set(SOURCE main.cpp)
add_executable(gui ${SOURCE})
target_link_libraries(gui Qt5::Widgets)
add_dependencies(gui client)
```

This is mostly concerned with adding the correct options so Qt's meta-object 
compiler can do its thing and we can locate the correct Qt libraries, however 
right down the bottom you'll notice that we create a new executable with 
`add_executable()`. This says our `gui` target has right now one source file, `main.cpp`. 
It also needs to link to `Qt5::Widgets` and depends on our `client` (the Rust library), which 
hasn't yet been configured.


## Building Rust with CMake

Next we need to create the Rust project.

```
$ cargo new client
```

To make it accessible from C++ we need to make sure `cargo` generates a 
dynamically linked library. This is just a case of tweaking our `Cargo.toml` to
tell `cargo` we're creating a `cdylib` instead of the usual library format. 

```toml
# client/Cargo.toml

[package]
name = "client"
version = "0.1.0"
authors = ["Michael Bryan <michaelfbryan@gmail.com>"]
description = "The business logic for a REST client"
repository = "https://github.com/Michael-F-Bryan/rust-ffi-guide"

[dependencies]

[lib]
crate-type = ["cdylib"]
```

If you then compile the project you'll see `cargo` build a shared object 
(`libclient.so`) instead of the normal `*.rlib` file.

```
$ cargo build
$ ls target/debug/
build  deps  examples  incremental  libclient.d  libclient.so  native
```

> **Note:** You don't *technically* need to make a dynamic library (`cdylib`) 
> for your Rust code to be callable from other languages. You can always use
> static linking with a `staticlib`, however that can be a bit more annoying to
> set up because you need to remember to link in a bunch of other things that
> the Rust standard library uses (mainly `libc` and the C runtime).
>
> With a dynamic library all the work for dependency resolution is handled by 
> the [loader] when your program gets loaded into memory on startup. Meaning 
> things should *Just Work*.

Now we know the Rust compiles natively with `cargo`, we need to hook it up to
`cmake`. We do this by writing a `CMakeLists.txt` in the `client/` directory.
As a general rule, you'll have one `CMakeLists.txt` for every "area" of your
code. This usually up being one per directory, but not always.

```cmake
# client/CMakeLists.txt

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CARGO_CMD cargo build)
    set(TARGET_DIR "debug")
else ()
    set(CARGO_CMD cargo build --release)
    set(TARGET_DIR "release")
endif ()

set(CLIENT_SO "${CMAKE_CURRENT_BINARY_DIR}/${TARGET_DIR}/libclient.so")

add_custom_target(client ALL
    COMMENT "Compiling client module"
    COMMAND CARGO_TARGET_DIR=${CMAKE_CURRENT_BINARY_DIR} ${CARGO_CMD} 
    COMMAND cp ${CLIENT_SO} ${CMAKE_CURRENT_BINARY_DIR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
set_target_properties(client PROPERTIES LOCATION ${CMAKE_CURRENT_BINARY_DIR})

add_test(NAME client_test 
    COMMAND cargo test
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

```

This is our first introduction to the difference between a debug and release
build. So we know whether to compile our program using different optimisation
levels and debug symbols, `cmake` will set a `CMAKE_BUILD_TYPE` variable
containing either `Debug` or `Release`. 

Here we're just using an `if` statement to set the `cargo` build command and
the target directory, then using those to add a custom target which will
first build the library, then copy the generated binary to the
`CMAKE_BINARY_DIR`.

For good measure, lets add a test (`client_test`) which lets `cmake` know how to
test our Rust module.

To make sure `cargo` puts all compiled artefacts in the correct spot within 
`build/`, we set the `CARGO_TARGET_DIR` environment variable while invoking the
`CARGO_CMD`. The compiled library is then copied from into 
`CMAKE_CURRENT_BINARY_DIR` and we set the `LOCATION` property on the overall 
target to be `CMAKE_CURRENT_BINARY_DIR`.

The purpose of that little dance is so that no matter what type of build 
(release or debug) we do, the compiled library will be in the same spot. We then
set the `client` target's `LOCATION` property so that anyone else who needs to 
use `client`'s outputs knows which directory they'll be in.

Now we know where the compiled `client` module will be, we can tell our `gui` to
link to it.

```diff
# gui/CMakeLists.txt

...

set(SOURCE main.cpp)
add_executable(gui ${SOURCE})
+ get_target_property(CLIENT_DIR client LOCATION)
target_link_libraries(gui Qt5::Widgets)
+ target_link_libraries(gui ${CLIENT_DIR}/libclient.so)
add_dependencies(gui client)
```

Now we can compile and run this basic program to make sure everything is 
working. You'll probably want to create a separate `build/` directory so you 
don't pollute the rest of the project with random build artefacts.

```
$ mkdir build && cd build
$ cmake ..
$ make
$ ./gui/gui
```


## Calling Rust from C++

So far we've just made sure everything compiles, however the C++ and Rust code 
are still completely independent. The next task is to check the Rust library is
linked to properly by calling a function from C++.

First we add a dummy function to the `lib.rs`.


```rust
#[no_mangle]
pub extern "C" fn hello_world() {
    println!("Hello World!");
}
```

There's a lot going on here, so lets step through it bit by bit.

The `#[no_mangle]` attribute indicates to the compiler that it shouldn't mangle 
the function's name during compilation. According to Wikipedia, 
[*name mangling*][mangle]:

> In compiler construction, name mangling (also called name decoration) is a
> technique used to solve various problems caused by the need to resolve unique
> names for programming entities in many modern programming languages.
>
> It provides a way of encoding additional information in the name of a
> function, structure, class or another datatype in order to pass more semantic
> information from the compilers to linkers.
>
> The need arises where the language allows different entities to be named
> with the same identifier as long as they occupy a different namespace (where
> a namespace is typically defined by a module, class, or explicit namespace
> directive) or have different signatures (such as function overloading).

**TL:DR;** *it's a way for compilers to generate multiple instances of a function
which accepts different types or parameters. Without it we wouldn't be able to
have things like generics or function overloading without name clashes.*

If this function is going to be called from C++ we need to specify the 
[calling convention] (the `extern "C"` bit). This tells the compiler low level
things like how arguments are passed between functions. By far the most common 
convention is to "just do what C does".

The rest of the function declaration should be fairly intuitive.

After recompiling (`cd build && cmake .. && make`) you can inspect the generated
binary using `nm` to make sure the `hello_world()` function is there.

```
$ nm libclient.so | grep ' T '
0000000000003330 T hello_world          <-- the function we created
00000000000096c0 T __rdl_alloc
00000000000098d0 T __rdl_alloc_excess
0000000000009840 T __rdl_alloc_zeroed
0000000000009760 T __rdl_dealloc
0000000000009a20 T __rdl_grow_in_place
0000000000009730 T __rdl_oom
0000000000009780 T __rdl_realloc
0000000000009950 T __rdl_realloc_excess
0000000000009a30 T __rdl_shrink_in_place
0000000000009770 T __rdl_usable_size
0000000000015ad0 T rust_eh_personality
```

The `nm` tool lists all the symbols in a binary as well as their addresses 
(the hex bit in the first column) and what type of symbol they are. All 
functions are in the **T**ext section of the binary, so you can use grep to view
only the exported functions.

Now we have a working library, why don't we make the GUI program less like a 
contrived example and more like a real-life application?

The first thing is to pull our main window out into its own source files.

```bash
$ touch gui/main_window.hpp gui/main_window.cpp
```

```diff
# gui/CMakeLists.txt

...

- set(SOURCE main.cpp)
+ set(SOURCE main_window.cpp main_window.hpp main.cpp)
add_executable(gui ${SOURCE})
```

```cpp
// gui/main_window.hpp

#include <QtWidgets/QMainWindow>
#include <QtWidgets/QPushButton>

class MainWindow : public QMainWindow {
  Q_OBJECT

public:
  MainWindow(QWidget *parent = nullptr);
private slots:
  void onClick();

private:
  QPushButton *button;
};
```

Here we've declared a `MainWindow` class which contains our trusty `QPushButton`
and has a single constructor and click handler.

We also need to fill out the `MainWindow` methods and hook up the button's 
`released` signal to our `onClick()` click handler.

```cpp
// gui/main_window.cpp

#include "main_window.hpp"

extern "C" {
void hello_world();
}

void MainWindow::onClick() { 
    // Call the `hello_world` function to print a message to stdout
    hello_world(); 
}

MainWindow::MainWindow(QWidget *parent) : QMainWindow(parent) {
  button = new QPushButton("Click Me", this);
  
  // Connect the button's `released` signal to `this->onClick()`
  connect(button, SIGNAL(released()), this, SLOT(onClick()));
}
```

Don't forget to update `main.cpp` to use the new `MainWindow`.

```cpp
// gui/main.cpp

#include "main_window.hpp"
#include <QtWidgets/QApplication>

int main(int argc, char **argv) {
  QApplication app(argc, argv);

  MainWindow mainWindow;
  mainWindow.show();

  app.exec();
}
```

Now when you compile and run `./gui`, "Hello World" wil be printed to the
console every time you click on the button.

If you got to this point then congratulations, you've just finished the most 
difficult part - getting everything to build!


[button]: http://doc.qt.io/qt-5/qpushbutton.html
[mangle]: https://en.wikipedia.org/wiki/Name_mangling
[calling convention]: https://en.wikipedia.org/wiki/Calling_convention
[loader]: https://en.wikipedia.org/wiki/Loader_(computing)