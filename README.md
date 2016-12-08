C++ WebSocket Server Workshop
=============================

**The completed source code for this workshop can be found here: <https://github.com/adamrehn/websocket-server-demo/>**.

This workshop demonstrates how to create a C++ server that communicates with a web client using the WebSocket protocol. The main skills covered are:

- Using the server functionality of the [WebSocket++](https://github.com/zaphoyd/websocketpp) WebSocket library
- Basic JSON serialisation using the [JsonCpp](https://github.com/open-source-parsers/jsoncpp) library
- Using the `asio::io_service` event loop functionality from [Boost.Asio](http://think-async.com/)
- Implementing the [Observer Pattern](https://en.wikipedia.org/wiki/Observer_pattern) using both `std::bind` and C++11 lambdas as callbacks
- Basic multi-threading using C++11 threads
- Communicating between event loops on multiple threads

The workshop is broken down into a series of parts:

- [Part 1: The basic client and server](./Part1.md)
- [Part 2: Adding JSON support for structured messages](./Part2.md)
- [Part 3: Initiating messages from the server](./Part3.md)
- [Part 4: Multiple threads and event loops](./Part4.md)
- [Part 5: Avoiding race conditions](./Part5.md)
