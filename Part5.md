C++ WebSocket Server Workshop
=============================

Part 5: Avoiding race conditions
--------------------------------

Now that we have multiple threads accessing the `WebsocketServer` class, we need to make sure that multiple threads can't access shared data simultaneously. Thanks to the way the event loops work, all of the network-related concurrency is already safe. The only member fields of the `WebsocketServer` class that need to be protected from concurrent access are the list of open connections, and the lists of event handlers:

```
vector<ClientConnection> openConnections;
vector<std::function<void(ClientConnection)>> connectHandlers;
vector<std::function<void(ClientConnection)>> disconnectHandlers;
map<string, vector<std::function<void(ClientConnection, const Json::Value&)>>> messageHandlers;
```

We could use mutexes to protect access to each of these objects individually, but there's actually a simpler way of protecting the event handler lists. The only external access points for these lists are the functions that register new event handlers. We can use the same trick that was previously used to ensure callbacks get run on the main thread, and post a lambda to the event loop for the networking thread, that will perform the actual modification of the lists. To do so, we need to create an event loop object that we have access to, and instruct WebSocket++ to use that instead of creating its own event loop internally. First we need to add a new member field to the class declaration:

```
asio::io_service eventLoop;
```

We can then modify the constructor of the `WebsocketServer` class to pass the new event loop to Asio. The last statement in the constructor:

```
this->endpoint.init_asio();
```

needs to be changed to become:

```
this->endpoint.init_asio(&(this->eventLoop));
```

Now that we have access to the event loop being used by the networking thread, we can modify the handler registration functions to use it:

```
//Registers a callback for when a client connects
template <typename CallbackTy>
void connect(CallbackTy handler)
{
    //Make sure we only access the handlers list from the networking thread
    this->eventLoop.post([this, handler]() {
        this->connectHandlers.push_back(handler);
    });
}

//Registers a callback for when a client disconnects
template <typename CallbackTy>
void disconnect(CallbackTy handler)
{
    //Make sure we only access the handlers list from the networking thread
    this->eventLoop.post([this, handler]() {
        this->disconnectHandlers.push_back(handler);
    });
}

//Registers a callback for when a particular type of message is received
template <typename CallbackTy>
void message(const string& messageType, CallbackTy handler)
{
    //Make sure we only access the handlers list from the networking thread
    this->eventLoop.post([this, messageType, handler]() {
        this->messageHandlers[messageType].push_back(handler);
    });
}
```

This is easier than using a mutex, because it requires no modification to the other functions that access the event handler lists, since we already know that those functions will be run on the networking thread. More importantly, the use of a mutex in those functions would have required a great deal of care to ensure that the critical section remained as small as possible, whilst still protecting the lists being iterated over.

The only remaining member field that requires protection from concurrent access is the list of open connections. Because the `numConnections` function accesses the list and returns a value to the caller, the lambda trick used earlier will not work, since event loop work items cannot return values. In this case, the simplest option is just to use a mutex. First, we add the necessary include:

```
#include <mutex>
``` 

We then add a mutex member field to the `WebsocketServer` class declaration:

```
std::mutex connectionListMutex;
```

The `onOpen`, `onClose`, `broadcastMessage`, and `numConnections` functions all need to be updated to lock the mutex while they are accessing or modifying the list of open connections:

```
size_t numConnections()
{
    //Prevent concurrent access to the list of open connections from multiple threads
    std::lock_guard<std::mutex> lock(this->connectionListMutex);
    
    return this->openConnections.size();
}

void broadcastMessage(const string& messageType, const Json::Value& arguments)
{
    //Prevent concurrent access to the list of open connections from multiple threads
    std::lock_guard<std::mutex> lock(this->connectionListMutex);
    
    for (auto conn : this->openConnections) {
        this->sendMessage(conn, messageType, arguments);
    }
}

void onOpen(ClientConnection conn)
{
    {
        //Prevent concurrent access to the list of open connections from multiple threads
        std::lock_guard<std::mutex> lock(this->connectionListMutex);
        
        //Add the connection handle to our list of open connections
        this->openConnections.push_back(conn);
    }
    
    //Invoke any registered handlers
    for (auto handler : this->connectHandlers) {
        handler(conn);
    }
}

void onClose(ClientConnection conn)
{
    {
        //Prevent concurrent access to the list of open connections from multiple threads
        std::lock_guard<std::mutex> lock(this->connectionListMutex);
        
        //Remove the connection handle from our list of open connections
        auto connVal = conn.lock();
        auto newEnd = std::remove_if(this->openConnections.begin(), this->openConnections.end(), [&connVal](ClientConnection elem)
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
        this->openConnections.resize(std::distance(openConnections.begin(), newEnd));
    }

    //Invoke any registered handlers
    for (auto handler : this->disconnectHandlers) {
        handler(conn);
    }
}
```

Now, all data structures that can be accessed from multiple threads are protected from concurrent access, thus preventing race conditions that could cause incorrect behaviour.
