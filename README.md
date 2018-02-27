# Adonis WebSocket
Adonis WebSocket library works on top of [WebSocket protocol](https://tools.ietf.org/html/rfc6455) and uses [ws](https://github.com/websockets/ws) library as the base to build upon.

This document describes the philosophy behind the library and shares the custom vocabulary added to the process.

## Terms Used
#### Packet
The packet sent from `client -> server` and `server -> client`. Each packet must have a type.

#### Channels
Channels makes it possible to separate the application concerns without creating a new TCP connection.

#### Topics
Topics are subscribed on a given channel. If channel name is static then the topic name will be the same as the channel name.

For example:

```js
Ws.channel('chat', function ({ socket }) {
  console.log(socket.topic)
  // will always be `chat`
})
```

If channel name has a wildcard, then multiple matching topics can be subscribed.

```js
Ws.channel('chat:*', function ({ socket }) {
})
```

This time topic name can be anything after `chat:`. It can be `chat:watercooler` or `chat:design` and so on.

The dynamic channel names makes it even simple to have a dynamic topics and a user can subscribe to any topic they want.

## Pure WebSockets
Adonis WebSocket uses pure WebSocket connection and never relies on polling. All of the browsers have support for WebSockets and there is no point in adding fallback layers.

By creating a pure WebSocket connection, we make it easier to scale apps horizontally, without relying on sticky sessions. Whereas with solutions like `socket.io` you need sticky sessions and it's even harder to use Node.js cluster module.

## Multiplexing
If you have worked with WebSockets earlier ( without any library ), you would have realized, there is no simple way to separate application concerns with in a single TCP connection.

If you want to have multiple channels like `chat` and `news`, the client have to open 2 separate TCP connections, which is waste of resources and overhead on server and client both.

Adonis introduces a layer of Channels, which uses a single TCP connection and uses messages a means of communicating within the channels.

If you are a consumer of the Adonis WebSocket library, you will get a clean abstraction to make use of channels.

If you are a developer creating a client library, then you will have to understand the concept of Packets and what they mean.

## Packets
Packets are a way to communicate between client and server using WebSocket messages.

For example: A packet to join a channel looks as follows.

```js
{
  t: 1,
  d: { topic: 'chat' }
}
```

1. The `t` is the packet code.
2. And `d` is the packet data. Each packet type has it's own data requirements.

Actions like `JOIN` and `LEAVE` are always acknowledged from the server with a successful acknowledgement or with an error.

Following is an example of `JOIN_ERROR`.

```js
{
  t: 4,
  d: {
    topic: 'chat',
    message: 'Topic has already been joined'
  }
}
```

Here's the list of packet types and their codes.

```js
{
    OPEN: 0,
    JOIN: 1,
    LEAVE: 2,
    JOIN_ACK: 3,
    JOIN_ERROR: 4,
    LEAVE_ACK: 5,
    LEAVE_ERROR: 6,
    EVENT: 7,
    PING: 8,
    PONG: 9
}
```

**Why numbers?** : Because it's less data to transfer.

## Out of the box cluster support
The server implementation of Adonis web socket supports Node.js cluster, without the need of any additional libraries.

It makes use of the `process` to send messages to each other.

## Contracts
By now as you know the messaging packets are used to build the channels and topics flow, below is the list of contracts **client and server** has to follow. 

1. **client**: `JOIN` packet must have a topic.
2. **server**: Server acknowledges the `JOIN` packet with `JOIN_ERROR` or `JOIN_ACK` packet. Both the packets will have the topic name in them.
3. **server**: Ensure a single TCP connection can join a given topic only for once.
4. **client**: `EVENT` packet must have a topic inside the message body, otherwise packet will be dropped by the server.
5. **server**: `EVENT` packet must have a topic inside the message body, otherwise packet will be dropped by the client too.

The `LEAVE` flow works same as the `JOIN` flow.

## Ping/Pong
Wish networks would have been reliable, but till then always be prepared for ungraceful disconnections.  Ping/Pong is a standard way for client to know that a server is alive and vice-versa.

In order to distribute load, AdonisJs never pings clients to find if they are alive or not, instead clients are expected to ping the server after given interval. **This approach is borrowed from Phoenix ( Elixir framework )**.

If a client fails to ping the server, their connection will be dropped after defined number of retries. Also for every `ping`, client will receive a `pong` from the server, which tells the client that the server is alive.

AdonisJs supports standard [ping/pong frames](https://tools.ietf.org/html/rfc6455#section-5.5.2) and if your client doesn't support sending these frames, then you can send a message with the `Packet type = PING`.

1. A single ping/pong game is played for a single TCP connection, there is no need to ping for each channel subscription.
2. When a connection is established, server sends an `OPEN` packet to the client, which contains the data to determine the ping interval.

```js
{
  t: 0,
  d: {
    serverInterval: 30000,
    serverAttempts: 3,
    clientInterval: 25000,
    connId: 'connection unique id',
    encoder: 'msgpack'
  }
}
```

1. All of the times are in milliseconds.
2. The `serverInterval` is the timer created on server to check the last ping time for a connection.
3. The `serverAttempts` is the number of attempts, server will perform before terminating client.
4. The `clientInterval` is timer client should set to ping the server. It will be always will be lower than the `serverInterval`.
5. The `connId` is the connection id on the server.
6. The `encoder` used by the server. Client has to use the same encoder to decode messages properly.

## Browser Support
WebSockets are supported on all major browsers, so there is no point of adding weird fallbacks.
https://caniuse.com/#feat=websockets

## Libraries

1. [@adonisjs/websocket-packet](http://npm.im/@adonisjs/websocket-packet) Library to **create** and **verify** packets.
2. [@adonisjs/msgpack-encoder](https://www.npmjs.com/package/@adonisjs/msgpack-encoder) The encoder based upon [msgpack](https://msgpack.org/index.html) used to encode/decode messages. The output of this library is smaller than `JSON.stringify` (which means less data to transfer)
3. [simple-message-reader](http://npm.im/simple-message-reader) Library for converting `Blob` and `Array buffer` to string.
4. [@adonisjs/websocket-client](http://npm.im/@adonisjs/websocket-client) A generic library to read the web socket messages on the browser and decode packages properly. You can make use of this library to build framework specific implementations.


## Example

Below is a basic example of how to implement a client in Javascript

```js
const filereader = require('simple-filereader')
const msgpack = require('msgpack-lite')
const packets = require('@adonisjs/websocket-packets')

const client = new window.WebSocket('ws://localhost:3000/adonis-ws')

client.onopen = function () {
  // TCP connection created
}

client.onerror = function () {
  // TCP connection error
}

client.onmessage = function (message) {
  filereader(message, function (error, payload) {
    const packet = msgpack.decode(payload)
    handlePacket(packet)
  })
}

function handlePacket (packet) {
  if (packets.isOpenPacket(packet)) {
    console.log('Server ack connection. Make channel subscriptions now')
  }
  
  if (packets.isJoinAck(packet)) {
    console.log('subscription created for %s', packet.d.topic)
  }
}
```

Above is the bare-bones example, the libraries for each platform will offer better abstractions.