C++ WebSocket Server Workshop
=============================

Part 4: Multiple threads and event loops
----------------------------------------

At the moment, the server has all of the functionality needed for communicating with clients, but is purely driven by events originating from clients. The main thread of the application is blocked by the networking event loop, and cannot accept any input from the user. To fix this, multiple threads are needed, as well as functionality for communicating between threads.


### Moving network communication to its own thread and accepting user input

Using C++11 threading functionality, it is easy to move all networking functionality to its own thread. First, we add the neccessary include:

```
#include <thread>
```

We can then update the `main` function to become:

```
int main (int argc, char* argv[])
{
    WebsocketServer server;
    
    //Start the networking thread
    std::thread serverThread([&server]() {
        server.run(8080);
    });
    
    //Accept user input and use it to generate messages
    string input;
    while (1)
    {
        //Read user input from stdin
        std::getline(std::cin, input);
        
        //Broadcast the input to all connected clients (is sent on the network thread)
        Json::Value payload;
        payload["input"] = input;
        server.broadcastMessage("userInput", payload);
    }
    
    //Block the main thread until the server thread has completed
    serverThread.join();
    
    return 0;
}
```

Now, the user can enter messages on standard input, which will be broadcast to all connected clients.


### Adding support for event handlers

Currently, network events are handled only inside the `WebsocketServer` class. For more flexibility, we can add functionality for registering event handlers from other code, and other threads. First, we add the necessary include:

```
#include <map>
using std::map;
```

We can then add three new member fields to the `WebsocketServer` class:

```
vector<std::function<void(ClientConnection)>> connectHandlers;
vector<std::function<void(ClientConnection)>> disconnectHandlers;
map<string, vector<std::function<void(ClientConnection, const Json::Value&)>>> messageHandlers;
```

We maintain lists of callbacks for connection and disconnection events, and a list of handlers for each different type of message, using the message type string as the key for the mapping. We use the `std::function` type for storing callbacks, since it can wrap any kind of callable object, including function pointers, function objects, lambdas, and the results of `std::bind`. To register new callbacks, we add three new template functions to the `WebsocketServer` class:

```
//Registers a callback for when a client connects
template <typename CallbackTy>
void connect(CallbackTy handler) {
    this->connectHandlers.push_back(handler);
}

//Registers a callback for when a client disconnects
template <typename CallbackTy>
void disconnect(CallbackTy handler) {
    this->disconnectHandlers.push_back(handler);
}

//Registers a callback for when a particular type of message is received
template <typename CallbackTy>
void message(const string& messageType, CallbackTy handler) {
    this->messageHandlers[messageType].push_back(handler);
}
```

We can now update the `WebsocketServer::onOpen` function to invoke any registered callbacks:

```
void onOpen(ClientConnection conn)
{
    //Add the connection handle to our list of open connections
    openConnections.push_back(conn);
    
    //Invoke any registered handlers
    for (auto handler : this->connectHandlers) {
        handler(conn);
    }
}
```

We can also update the `WebsocketServer::onClose` function to invoke registered callbacks:

```
void onClose(ClientConnection conn)
{
    //Remove the connection handle from our list of open connections
    auto connVal = conn.lock();
    auto newEnd = std::remove_if(openConnections.begin(), openConnections.end(), [&connVal](ClientConnection elem)
    {
        //If the pointer has expired, remove it from the vector
        if (elem.expired() == true) {
            return true;
        }
        
        //If the pointer is still valid, compare it to the handle for the closed connection
        auto elemVal = elem.lock();
        if (elemVal.get() == connVal.get()) {
            return true;
        }
        
        return false;
    });
    
    //Truncate the connections vector to erase the removed elements
    openConnections.resize(std::distance(openConnections.begin(), newEnd));
    
    //Invoke any registered handlers
    for (auto handler : this->disconnectHandlers) {
        handler(conn);
    }
}
```

Finally, we can update the `WebsocketServer::onMessage` to invoke the registered callbacks for the specific message type received:

```
void onMessage(ClientConnection conn, WebsocketEndpoint::message_ptr msg)
{
    //Validate that the incoming message contains valid JSON
    Json::Value messageObject = WebsocketServer::parseJson(msg->get_payload());
    if (messageObject.isNull() == false)
    {
        //Validate that the JSON object contains the message type field
        if (messageObject.isMember("__MESSAGE__"))
        {
            //Extract the message type and remove it from the payload
            std::string messageType = messageObject["__MESSAGE__"].asString();
            messageObject.removeMember("__MESSAGE__");
            
            //If any handlers are registered for the message type, invoke them
            auto& handlers = this->messageHandlers[messageType];
            for (auto handler : handlers) {
                handler(conn, messageObject);
            }
        }
    }
}
```


### Multiple event loops

By default, any callbacks registered with the `WebsocketServer` class will be invoked in the event loop of the network thread. This means that any long-running processing in these callbacks will block network connectivity until they complete. For example, if we were to add the following code to `main`, all of the registered callbacks would be run on the networking thread:

```
server.connect([&server](ClientConnection conn)
{
    std::clog << "Connection opened." << std::endl;
    
    //Send a hello message to the client
    server.sendMessage(conn, "hello", Json::Value());
});

server.disconnect([](ClientConnection conn) {
    std::clog << "Connection closed." << std::endl;
});

server.message("message", [&server](ClientConnection conn, const Json::Value& args)
{
    std::clog << "Received message of type \"message\"." << std::endl;
    std::clog << "Message payload:" << std::endl;
    for (auto key : messageObject.getMemberNames()) {
        std::clog << "\t" << key << ": " << messageObject[key].asString() << std::endl;
    }
    
    //Echo the message pack to the client
    server.sendMessage(conn, "message", args);
});
```

At the moment, the main thread is blocked by the user input loop, and does not have an event loop to which work can be posted by other threads. To solve this, we can add an Asio `io_service` event loop to the main thread, and move the user input loop to its own thread:

```
//Create an event loop for the main thread
asio::io_service mainEventLoop;

//Start a keyboard input thread that reads from stdin
std::thread inputThread([&server]()
{
    string input;
    while (1)
    {
        //Read user input from stdin
        std::getline(std::cin, input);
        
        //Broadcast the input to all connected clients (is sent on the network thread)
        Json::Value payload;
        payload["input"] = input;
        server.broadcastMessage("userInput", payload);
    }
});

//Start the event loop for the main thread
asio::io_service::work work(mainEventLoop);
mainEventLoop.run();
```

Now, the user input blocks only its own background thread. Because the WebSocket++ endpoint uses its own `io_service` internally, all requests to transmit data to clients will actually be carried out on the network thread's event loop, which means the user input thread won't be blocked by data transmission. The event loop for the main thread is ready to have work posted to it from other threads.

We can implement callbacks that run on the main thread's event loop by utilising nested lambdas:

```
//Register our network callbacks, ensuring the logic is run on the main thread's event loop
server.connect([&mainEventLoop, &server](ClientConnection conn)
{
    mainEventLoop.post([conn, &server]()
    {
        std::clog << "Connection opened." << std::endl;
        
        //Send a hello message to the client
        server.sendMessage(conn, "hello", Json::Value());
    });
});

server.disconnect([&mainEventLoop, &server](ClientConnection conn)
{
    mainEventLoop.post([conn, &server]() {
        std::clog << "Connection closed." << std::endl;
    });
});

server.message("message", [&mainEventLoop, &server](ClientConnection conn, const Json::Value& args)
{
    mainEventLoop.post([conn, args, &server]()
    {
        std::clog << "Received message of type \"message\"." << std::endl;
        std::clog << "Message payload:" << std::endl;
        for (auto key : messageObject.getMemberNames()) {
            std::clog << "\t" << key << ": " << messageObject[key].asString() << std::endl;
        }
        
        //Echo the message pack to the client
        server.sendMessage(conn, "message", args);
    });
});
```

Now, the callbacks that are invoked on the networking thread post the nested lambdas to the main thread's event loop, ensuring the actual processing is performed on the main thread.
