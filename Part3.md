C++ WebSocket Server Workshop
=============================

Part 3: Initiating messages from the server
-------------------------------------------


### Echoing received messages back to the client

To add functionality for sending messages to the client from the server, we add the following function to the `WebsocketServer` class:

```
void sendMessage(ClientConnection conn, const string& messageType, const Json::Value& arguments)
{
    //Copy the argument values, and bundle the message type into the object
    Json::Value messageData = arguments;
    messageData["__MESSAGE__"] = messageType;
    
    //Send the JSON data to the client
    this->endpoint.send(conn, WebsocketServer::stringifyJson(messageData), websocketpp::frame::opcode::text);
}
```

We can now echo messages back to the client when we receive them, by adding the following code to the `WebsocketServer::onMessage` function:

```
this->sendMessage(conn, messageType, messageObject);
```


### Broadcasting messages to all connected clients

At the moment, the server does not keep track of which clients are currently connected. To facilitate broadcasting messages to all clients, we need to add functionality to maintain a list of connected clients. First, we need to add the following includes:

```
#include <vector>
using std::vector;
```

We can then add a member field to the `WebsocketServer` class to hold the list of connections:

```
vector<ClientConnection> openConnections;
```

To update the list, we need to modify the `WebsocketServer::onOpen` and `WebsocketServer::onClose` functions to become:

```
void WebsocketServer::onOpen(ClientConnection conn)
{
    //Add the connection handle to our list of open connections
    openConnections.push_back(conn);
}

void WebsocketServer::onClose(ClientConnection conn)
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
}
```

The convoluted code in `WebsocketServer::onClose` is required due to the use of a `std::weak_ptr<void>` for the connection handle type, which does not facilitate comparison with other handles without being upgraded to a `std::shared_ptr` instance first.

Now that we have a list of connected clients, we can easily implement functionality to broadcast messages to all of them at once:

```
void broadcastMessage(const string& messageType, const Json::Value& arguments)
{
    for (auto conn : this->openConnections) {
        this->sendMessage(conn, messageType, arguments);
    }
}
```
