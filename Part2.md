C++ WebSocket Server Workshop
=============================

Part 2: Adding JSON support for structured messages
---------------------------------------------------

Instead of using purely unstructured messages, we want to modify the client and server so that messages must specify a type, and can provide additional data in the form of an object containing key/value pairs. This provides the flexibility to support any number of arbitrary message types, each with a specific set of arguments, whilst being far more structured than raw strings.


### Adding JSON support to the server

First, we need to include the JsonCpp library header at the top of the server source code, near the WebSocket++ includes. We also need to include the C++ `<string>` header, since we will be using strings in our JSON processing code:

```
#include <json/json.h>
#include <string>
using std::string;
```

Next, we add two functions to the `WebsocketServer` class, to handle parsing and generating JSON strings:

```
Json::Value parseJson(const string& json)
{
    Json::Value root;
    Json::Reader reader;
    reader.parse(json, root);
    return root;
}

string stringifyJson(const Json::Value& val)
{
    //When we transmit JSON data, we omit all whitespace
    Json::StreamWriterBuilder wbuilder;
    wbuilder["commentStyle"] = "None";
    wbuilder["indentation"] = "";
    
    return Json::writeString(wbuilder, val);
}
```

Now, we can update the `WebsocketServer::onMessage` function to parse incoming messages as JSON:

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
            
            //Output the details
            std::clog << "Received message of type \"" << messageType << "\"." << std::endl;
            std::clog << "Message payload:" << std::endl;
            for (auto key : messageObject.getMemberNames()) {
                std::clog << "\t" << key << ": " << messageObject[key].asString() << std::endl;
            }
        }
    }
}
```

Messages that are not well-formed JSON with a `__MESSAGE__` field will simply be ignored. Well-formed messages are parsed and then handled accordingly (at the moment, just printing debug output.)


### Adding JSON support to the client

On the client side, we add a new class to wrap the `SimpleWebsocket` class and provide JSON support:

```
function SocketWrapper(init)
{
    this.socket = new SimpleWebsocket(init);
    this.messageHandlers = {};
    
    var that = this;
    this.socket.on('data', function(data)
    {
        //Extract the message type
        var messageData = JSON.parse(data);
        var messageType = messageData['__MESSAGE__'];
        delete messageData['__MESSAGE__'];
        
        //If any handlers have been registered for the message type, invoke them
        if (that.messageHandlers[messageType] !== undefined)
        {
            for (index in that.messageHandlers[messageType]) {
                that.messageHandlers[messageType][index](messageData);
            }
        }
    });
}

SocketWrapper.prototype.on = function(event, handler)
{
    //If a standard event was specified, forward the registration to the socket's event emitter
    if (['connect', 'close', 'data', 'error'].indexOf(event) != -1) {
        this.socket.on(event, handler);
    }
    else
    {
        //The event names a message type
        if (this.messageHandlers[event] === undefined) {
            this.messageHandlers[event] = [];
        }
        this.messageHandlers[event].push(handler);
    }
}

SocketWrapper.prototype.send = function(message, payload)
{
    //Copy the values from the payload object, if one was supplied
    var payloadCopy = {};
    if (payload !== undefined && payload !== null)
    {
        var keys = Object.keys(payload);
        for (index in keys)
        {
            var key = keys[index];
            payloadCopy[key] = payload[key];
        }
    }
    
    payloadCopy['__MESSAGE__'] = message;
    this.socket.send(JSON.stringify(payloadCopy));
}
```

The wrapped `on` function allows us to add event handlers for specific message types, in addition to the default events provided by the underlying `SimpleWebsocket` instance. The `send` function takes a message type and optional arguments object, and handles the JSON serialisation required when sending messages to the server. The message-specific event handler mechanism parses incoming JSON messages and passes the details to the registered handlers.

To use the new functionality, we need to change the line

```
var socket = new SimpleWebsocket("ws://127.0.0.1:8080");
```

to become:

```
var socket = new SocketWrapper("ws://127.0.0.1:8080");
```

To provide the user with input fields for both the message type and arguments, we need to update the contents of the `<body>` tag to become:

```
<body>
    <input type="text" id="message" value="message">
    <input type="text" id="args" value='{"a":1, "b":2, "c":3}'>
    <button id="send">Send Message</button>
    <pre id="output"></pre>
</body>
```

We can then modify the button click handler from 

```
$('#send').click(function() {
    socket.send($('#message').val());
});
```

to become:

```
$('#send').click(function() {
    socket.send($('#message').val(), JSON.parse($('#args').val()));
});
```

For this to work correctly, the text entered into the `args` input field must be in JSON format. Otherwise, the `JSON.parse()` function will throw an error. Of course, in an actual application, these values would be generated programmatically, rather than taken directly from user input.
