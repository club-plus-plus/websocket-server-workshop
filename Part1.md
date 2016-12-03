C++ WebSocket Server Workshop
=============================

Part 1: The basic client and server
-----------------------------------


### Dependencies

We will be using the following libraries:

- [WebSocket++](https://github.com/zaphoyd/websocketpp)
- [Asio](http://think-async.com/)
- [JsonCpp](https://github.com/open-source-parsers/jsoncpp)

Both WebSocket++ and Asio are header-only libraries, which mean they can simply be downloaded without the need for compilation. JsonCpp consists of a single source file and a single header file, and can either be compiled into a library and linked against, or simply added to the source tree of the application and compiled that way.


### The server

The basic server code is based on the `echo_server` example code from the WebSocket++ examples directory: <https://github.com/zaphoyd/websocketpp/blob/master/examples/echo_server/echo_server.cpp>

```
//We need to define this when using the Asio library without Boost
#define ASIO_STANDALONE

#include <websocketpp/config/asio_no_tls.hpp>
#include <websocketpp/server.hpp>

//Provides std::bind and std::placeholders
#include <functional>

typedef websocketpp::server<websocketpp::config::asio> WebsocketEndpoint;
typedef websocketpp::connection_hdl ClientConnection;

class WebsocketServer
{
    public:
        
        WebsocketServer()
        {
            //Wire up our event handlers
            this->endpoint.set_open_handler(std::bind(&WebsocketServer::onOpen, this, std::placeholders::_1));
            this->endpoint.set_close_handler(std::bind(&WebsocketServer::onClose, this, std::placeholders::_1));
            this->endpoint.set_message_handler(std::bind(&WebsocketServer::onMessage, this, std::placeholders::_1, std::placeholders::_2));
            
            //Initialise the Asio library
            this->endpoint.init_asio();
        }
        
        void run(int port)
        {
            //Listen on the specified port number and start accepting connections
            endpoint.listen(port);
            endpoint.start_accept();
            
            //Start the Asio event loop
            endpoint.run();
        }
        
    protected:
        
        void onOpen(ClientConnection conn) {
            std::clog << "Client connected." << std::endl;
        }
        
        void onClose(ClientConnection conn) {
            std::clog << "Client disconnected." << std::endl;
        }
        
        void onMessage(ClientConnection conn, WebsocketEndpoint::message_ptr msg) {
            std::clog << "Received message: \"" << msg->get_payload() << "\"" << std::endl;
        }
        
        WebsocketEndpoint endpoint;
};

int main (int argc, char* argv[])
{
    WebsocketServer server;
    server.run(8080);
    
    return 0;
}
```

There are a number of important things happening in this code:

- The WebSocket++ library supports a number of configurations. The typedef of `websocketpp::server<websocketpp::config::asio>` indicates that we are using Asio as the underlying network implementation. Asio is available as part of the [Boost](http://www.boost.org/) project, or standalone. The standalone version is the easiest option.
- The Asio library provides an event loop implementation, in the form of the `asio::io_service` class. The event loop is used by WebSocket++ internally to handle all network operations asynchronously.
- To handle network events, we must register callbacks for the events we are interested in. The full list of events supported by WebSocket++ can be found here: <https://www.zaphoyd.com/websocketpp/manual/reference/handler-list>.
- When registering callbacks for the open, close, and message handlers, we utilise the `std::bind` function. This allows us to create a callable object that encapsulates a function pointer, with pre-bound values for some of the function parameters, and placeholders for the parameters that will be filled-in when the object is called.
- Once the Asio event loop is started, it blocks the thread until it is explicitly stopped. The registered callbacks will be invoked as part of this event loop.


### The client

A basic client is simple to implement using the [jQuery](https://jquery.com/) and [simple-websocket](https://github.com/feross/simple-websocket) libraries:

```
<!doctype html>
<html>
    <head>
        <title>WebSocket Client Test</title>
        
        <script type="text/javascript" src="https://code.jquery.com/jquery-3.1.1.min.js"></script>
        <script type="text/javascript" src="simplewebsocket.min.js"></script>
        <script type="text/javascript">
            
            function log(text)
            {
                var outputElem = $('#output');
                outputElem.text( outputElem.text() + text + '\n' );
            }
            
            $(document).ready(function()
            {
                var socket = new SimpleWebsocket("ws://127.0.0.1:8080");
                
                socket.on('connect', function() {
                    log("socket is connected!");
                });
                
                socket.on('data', function(data) {
                    log('got message: ' + data)
                });
                
                socket.on('close', function() {
                    log("socket is disconnected!");
                });
                
                socket.on('error', function(err) {
                    log("Error: " + err);
                });
                
                $('#send').click(function() {
                    socket.send($('#message').val());
                });
            });
            
        </script>
    </head>
    
    <body>
        <input type="text" id="message" value=""><button id="send">Send Message</button>
        <pre id="output"></pre>
    </body>
</html>
```

When the page loads, a WebSocket is created and a connection to the server is established. Event handlers for all of the socket's events simply log information to an output `<pre>` tag, and a text field and button allow the user to enter a message and send it to the server.
