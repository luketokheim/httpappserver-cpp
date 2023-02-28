# Http Microservice for C++

[![sanitizer](https://github.com/luketokheim/httpmicroservice-cpp/actions/workflows/sanitizer.yaml/badge.svg)](https://github.com/luketokheim/httpmicroservice-cpp/actions/workflows/sanitizer.yaml)
[![test](https://github.com/luketokheim/httpmicroservice-cpp/actions/workflows/test.yaml/badge.svg)](https://github.com/luketokheim/httpmicroservice-cpp/actions/workflows/test.yaml)
[![tidy](https://github.com/luketokheim/httpmicroservice-cpp/actions/workflows/tidy.yaml/badge.svg)](https://github.com/luketokheim/httpmicroservice-cpp/actions/workflows/tidy.yaml)

Run your C++ function as a containerized web service.

## Quick Start

A minimal service needs a request handler.

```cpp
asio::awaitable<usrv::response> hello_world(usrv::request req)
{
    usrv::response res{http::status::ok, req.version()};
    res.set(http::field::content_type, "text/plain");
    res.body() = "Hello World!";

    co_return res;
}
```

And a main function.

```cpp
#include <httpmicroservice/service.hpp>

namespace asio = boost::asio;
namespace http = boost::beast::http;
namespace usrv = httpmicroservice;

int main()
{
    // Listen on port 8080 and route all HTTP requests to the hello_world handler
    usrv::run(8080, hello_world);

    return 0;
}
```

Asio has excellent docs. Refer to those for more details on
[Basic Asio Anatomy](https://think-async.com/Asio/asio-1.24.0/doc/asio/overview/basics.html)
and [C++20 Coroutines Support](https://think-async.com/Asio/asio-1.24.0/doc/asio/overview/composition/cpp20_coroutines.html).

## Docker

Build a Docker image that runs the [Hello World](examples/hello.cpp) web service.

```console
docker build -t httpmicroservice-cpp .
```

Run the container.

```console
docker run --rm -p 8080:8080 httpmicroservice-cpp
```

The image is based on the empty Docker "scratch" image and only contains the
single binary [Hello World](examples/hello.cpp) example server.

## Design

The basic idea is that you use this library to run your C++ function in response
to an HTTP request. The library provides the server functionality and handles the
networking and protocol aspects for you.

The service runs one thread for all network I/O. For blocking or long running
synchronous tasks inside your C++ function your may want to provide a worker
thread pool to keep the main event loop running and processing other requests.

This service is intended to run behind a reverse proxy that terminates TLS and
maps requests to this application. I am using it on Google Cloud Run but other
containerized environments will work.

Since this service assumes it is running behind a reverse proxy there are some
features that I omitted.

- No SSL/TLS support
- No request or keep alive timeouts
- No request target resource to handler mapping
- No static content or nice error pages

## Async

The server uses only asynchronous operations. If you have used Asio before then
this main function will look more familiar.

```cpp
int main()
{
    asio::io_context ioc;

    // Listen on port 8080 and route all HTTP requests to the hello_world handler
    usrv::async_run(ioc, 8080, hello_world);

    // Run event processing loop
    ioc.run();

    return 0;
}
```

Most of my usage is with a single thread calling io_context::run().

## Requirements

This project is a C++20 library that uses coroutines for network I/O. The use
of coroutines is inspired by the [Talking Async Ep1: Why C++20 is the Awesomest
Language for Network Programming](https://youtu.be/icgnqFM-aY4) video by Chris
Kohlhoff.

- [Coroutines](https://en.cppreference.com/w/cpp/language/coroutines) support from a modern compiler
- [Asio](https://think-async.com/Asio/) for network I/O
- [Boost.Beast](https://github.com/boostorg/beast) to parse HTTP requests and form responses
- [Catch2](https://github.com/catchorg/Catch2) to run tests for continuous integration

For production use I recommend using io_uring (liburing-dev) on Linux if
available. It is enabled by default with the ENABLE_IO_URING CMake option if
liburing is found. The Docker and Continuous Deployment (CD) builds do not
install that library to maximize compatibility.

Cloud Run [second generation](https://cloud.google.com/run/docs/about-execution-environments)
execution environment supports io_uring but the managed container runtimes on
AWS (App Runner) and Azure (Container Apps) do not.

## Compilers

This project requires C++20 support for coroutines. It runs on Windows, macOS,
and Linux.

- Microsoft Visual Studio 2022
- Clang 13
- G++ 10

## Package managers

This project uses the [conan](https://conan.io/) C++ package manager to build
for Continuous Integration (CI) and its Docker images.

## Build

Create a build folder and install dependencies with the package manager.

```console
conan install . --output-folder=build --build=missing
```

Use the toolchain file created by the package manager so cmake can locate
libraries with [find_package](https://cmake.org/cmake/help/latest/command/find_package.html).

```console
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=conan_toolchain.cmake
cmake --build . --config=Release
```

Run tests.

```console
ctest -C Release
```
