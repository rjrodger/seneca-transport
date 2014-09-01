seneca-transport - a [Seneca](http://senecajs.org) plugin
======================================================

## Seneca Transport Plugin

This plugin provides the HTTP and TCP transport channels for
micro-service messages. It's a built-in dependency of the Seneca
module, so you don't need to include it manually. You use this plugin
to wire up your micro-services so that they can talk to each other.

[![Build Status](https://travis-ci.org/rjrodger/seneca-transport.png?branch=master)](https://travis-ci.org/rjrodger/seneca-transport)

[![NPM](https://nodei.co/npm/seneca-transport.png)](https://nodei.co/npm/seneca-transport/)
[![NPM](https://nodei.co/npm-dl/seneca-transport.png)](https://nodei.co/npm-dl/seneca-transport/)

For a gentle introduction to Seneca itself, see the
[senecajs.org](http://senecajs.org) site.

If you're using this plugin module, feel free to contact me on twitter if you
have any questions! :) [@rjrodger](http://twitter.com/rjrodger)

Current Version: 0.2.3

Tested on: Seneca 0.5.19, Node 0.10.29


### Install

This plugin module is included in the main Seneca module:

```sh
npm install seneca
```

To install separately, use:

```sh
npm install seneca-transport
```


## Quick Example

Let's do everything in one script to begin with. You'll define a
simple Seneca plugin that returns the hex value of color words. In
fact, all it can handle is the color red!

You define the action pattern _color:red_, which aways returns the
result <code>{hex:'#FF0000'}</code>. You're also using the name of the
function _color_ to define the name of the plugin (see [How to write a
Seneca plugin](http://senecajs.org)).

```js
function color() {
  this.add( 'color:red', function(args,done){
    done(null, {hex:'#FF0000'});
  })
}
```

Now, let's create a server and client. The server Seneca instance will
load the _color_ plugin and start a web server to listen for inbound
messages. The client Seneca instance will submit a _color:red_ message
to the server.


```js
var seneca = require('seneca')
      
seneca()
  .use(color)
  .listen()

seneca()
  .client()
  .act('color:red')
```

You can create multiple instances of Seneca inside the same Node.js
process. They won't interfere with each other, but they will share
external options from configuration files or the command line.

If you run the full script (full source is in
[readme-color.js](https://github.com/rjrodger/seneca-transport/blob/master/test/readme-color.js)),
you'll see the standard Seneca startup log messages, but you won't see
anything that tells you what the _color_ plugin is doing since this
code doesn't bother printing the result of the action. Let's use a
filtered log to output the inbound and outbound action messages from
each Seneca instance so we can see what's going on. Run the script with:

```sh
node readme-color.js --seneca.log=type:act,regex:color:red
```

This log filter restricts printed log entries to those that report
inbound and outbound actions, and further, to those log lines that
match the regular expression <code>/color:red/</code>. Here's what
you'll see:

```sh
[TIME] vy../..15/- DEBUG act -     - IN  485n.. color:red {color=red}   CLIENT 
[TIME] ly../..80/- DEBUG act color - IN  485n.. color:red {color=red}   f2rv..
[TIME] ly../..80/- DEBUG act color - OUT 485n.. color:red {hex=#FF0000} f2rv..
[TIME] vy../..15/- DEBUG act -     - OUT 485n.. color:red {hex=#FF0000} CLIENT
```

The second field is the identifier of the Seneca instance. You can see
that first the client (with an identifier of _vy../..15/-_) sends the
message <code>{color=red}</code>. The message is sent over HTTP to the
server (which has an identifier of _ly../..80/-_). The server performs the
action, generating the result <code>{hex=#FF0000}</code>, and sending
it back.

The third field, <code>DEBUG</code>, indicates the log level. The next
field, <code>act</code> indicates the type of the log entry. Since
you specified <code>type:act</code> in the log filter, you've got a
match!

The next two fields indicate the plugin name and tag, in this case <code>color
-</code>. The plugin is only known on the server side, so the client
just indicates a blank entry with <code>-</code>. For more details on
plugin names and tags, see [How to write a Seneca
plugin](http://senecajs.org).

The next field (also known as the _case_) is either <code>IN</code> or
<code>OUT</code>, and indicates the direction of the message. If you
follow the flow, you can see that the message is first inbound to the
client, and then inbound to the server (the client sends it
onwards). The response is outbound from the server, and then outbound
from the client (back to your own code). The field after that,
<code>485n..</code>, is the message identifier. You can see that it
remains the same over multiple Seneca instances. This helps you to
debug message flow.

The next two fields show the action pattern of the message
<code>color:red</code>, followed by the actual data of the request
message (when inbound), or the response message (when outbound).

The last field <code>f2rv..</code> is the internal identifier of the
action function that acts on the message. On the client side, there is
no action function, and this is indicated by the <code>CLIENT</code>
marker. If you'd like to match up the action function identifier to
message executions, add a log filter to see them:

```sh
node readme-color.js --seneca.log=type:act,regex:color:red \
--seneca.log=plugin:color,case:ADD
[TIME] ly../..80/- DEBUG plugin color - ADD f2rv.. color:red
[TIME] vy../..15/- DEBUG act    -     - IN  485n.. color:red {color=red}   CLIENT 
[TIME] ly../..80/- DEBUG act    color - IN  485n.. color:red {color=red}   f2rv..
[TIME] ly../..80/- DEBUG act    color - OUT 485n.. color:red {hex=#FF0000} f2rv..
[TIME] vy../..15/- DEBUG act    -     - OUT 485n.. color:red {hex=#FF0000} CLIENT
```

The filter <code>plugin:color,case:ADD</code> picks out log entries of
type _plugin_, where the plugin has the name _color_, and where the
_case_ is ADD. These entries indicate the action patterns that a
plugin has registered. In this case, there's only one, _color:red_.

You've run this example in a single Node.js process up to now. Of
course, the whole point is to run it a separate processes! Let's do
that. First, here's the server:

```js
function color() {
  this.add( 'color:red', function(args,done){
    done(null, {hex:'#FF0000'});
  })
}

var seneca = require('seneca')
      
seneca()
  .use(color)
  .listen()
```

Run this in one terminal window with:

```sh
$ node readme-color-service.js --seneca.log=type:act,regex:color:red
```

And on the client side:

```js
var seneca = require('seneca')
      
seneca()
  .client()
  .act('color:red')
```

And run with:

```sh
$ node readme-color-client.js --seneca.log=type:act,regex:color:red
```

You'll see the same log lines as before, just split over the two processes. The full source code is the [test folder](https://github.com/rjrodger/seneca-transport/tree/master/test).


## Non-Seneca Clients

The default transport mechanism for messages is HTTP. This means you can communicate easily with a Seneca micro-service from other platforms. By default, the <code>listen</code> method starts a web server on port 10101, listening on all interfaces. If you run the _readme-color-service.js_ script again (as above), you can talk to it by _POSTing_ JSON data to the <code>/act</code> path. Here's an example using the command line _curl_ utility.

```sh
$ curl -d '{"color":"red"}' http://localhost:10101/act
{"hex":"#FF0000"}
```

If you dump the response headers, you'll see some additional headers that give you contextual information. Let's use the <code>-v</code> option of _curl_ to see them:

```sh
$ curl -d '{"color":"red"}' -v http://localhost:10101/act
...
* Connected to localhost (127.0.0.1) port 10101 (#0)
> POST /act HTTP/1.1
> User-Agent: curl/7.30.0
> Host: localhost:10101
> Accept: */*
> Content-Length: 15
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 15 out of 15 bytes
< HTTP/1.1 200 OK
< Content-Type: application/json
< Cache-Control: private, max-age=0, no-cache, no-store
< Content-Length: 17
< seneca-id: 9wu80xdsn1nu
< seneca-kind: res
< seneca-origin: curl/7.30.0
< seneca-accept: sk5mjwcxxpvh/1409222334824/-
< seneca-time-client-sent: 1409222493910
< seneca-time-listen-recv: 1409222493910
< seneca-time-listen-sent: 1409222493910
< Date: Thu, 28 Aug 2014 10:41:33 GMT
< Connection: keep-alive
< 
* Connection #0 to host localhost left intact
{"hex":"#FF0000"}
```

You can get the message identifier from the _seneca-id_ header, and
the identifier of the Seneca instance from _seneca-accept_.

There are two structures that the submitted JSON document can take:

   * Vanilla JSON containing your request message, plain and simple, as per the example above,
   * OR: A JSON wrapper containing the client details along with the message data.

The JSON wrapper follows the standard form of Seneca messages used in
other contexts, such as message queue transports. However, the simple
vanilla format is perfectly valid and provided explicitly for
integration. The wrapper format is described below.

If you need Seneca to listen on a particular port or host, you can
specify these as options to the <code>listen</code> method. Both are
optional.

```js
seneca()
  .listen( { host:'192.168.1.2', port:80 } )
```

On the client side, either with your own code, or the Seneca client,
you'll need to use matching host and port options.

```bash
$ curl -d '{"color":"red"}' http://192.168.1.2:80/act
```

```js
seneca()
  .client( { host:'192.168.1.2', port:80 } )
```

You can also set the host and port via the Seneca options facility. When
using the options facility, you are setting the default options for
all message transports. These can be overridden by arguments to individual
<code>listen</code> and <code>client</code> calls.

Let's run the color example again, but with a different port. On the server-side:

```sh
$ node readme-color-service.js --seneca.log=type:act,regex:color:red \
  --seneca.options.transport.port=8888
```

And the client-side:

```sh
curl -d '{"color":"red"}' -v http://localhost:8888/act
```
OR

```sh
$ node readme-color-client.js --seneca.log=type:act,regex:color:red \
  --seneca.options.transport.port=8888
```

## Using the TCP Channel

Also included in this plugin is a TCP transport mechanism. The HTTP
mechanism offers easy integration, but it is necessarily slower. The
TCP transport open a direct TCP connection to the server. The
connection remains open, avoid connection overhead for each
message. The client side of the TCP transport will also attempt to
reconnect if the connection breaks, providing fault tolerance for
server restarts.

To use the TCP transport, specify a _type_ property to the
<code>listen</code> and <code>client</code> methods, and give the
value _tcp_. Here's the single script example again:


```js
seneca()
  .use(color)
  .listen({type:'tcp'})

seneca()
  .client({type:'tcp'})
  .act('color:red')
```

The full source code is in the
[readme-color-tcp.js](https://github.com/rjrodger/seneca-transport/blob/master/test/readme-color-tcp.js)
file. When you run this script it would be great to verify that the right transport channels are being created. You'd like to see the configuration, and any connections that occur. By default, this information is printed with a log level of _INFO_, so you will see it if you don't use any log filters.

Of course, we are using a log filter. So let's add another one to print the connectio details so we can sanity check the system. We want to print any log entries with a log level of _INFO_. Here's the command:

```sh
$ node readme-color-tcp.js --seneca.log=level:INFO \
  --seneca.log=type:act,regex:color:red
```

This produces the log output:

```sh
[TIME] 6g../..49/- INFO  hello  Seneca/0.5.20/6g../..49/-
[TIME] f1../..79/- INFO  hello  Seneca/0.5.20/f1../..79/-
[TIME] f1../..79/- DEBUG act    -         - IN  wdfw.. color:red {color=red} CLIENT 
[TIME] 6g../..49/- INFO  plugin transport - ACT b01d.. listen open {type=tcp,host=0.0.0.0,port=10201,...}
[TIME] f1../..79/- INFO  plugin transport - ACT nid1.. client {type=tcp,host=0.0.0.0,port=10201,...} any
[TIME] 6g../..49/- INFO  plugin transport - ACT b01d.. listen connection {type=tcp,host=0.0.0.0,port=10201,...} remote 127.0.0.1 52938
[TIME] 6g../..49/- DEBUG act    color     - IN  bpwi.. color:red {color=red} mcx8i4slu68z UNGATE
[TIME] 6g../..49/- DEBUG act    color     - OUT bpwi.. color:red {hex=#FF0000} mcx8i4slu68z
[TIME] f1../..79/- DEBUG act    -         - OUT wdfw.. color:red {hex=#FF0000} CLIENT
```

The inbound and outbound log entries are as before. In addition, you
can see the _INFO_ level entries. At startup, Seneca logs a "hello"
entry with the identifier of the current instance execution. This
identifier has the form:
<code>Seneca/[12-random-chars]/[timestamp]/[tag]</code>.  This
identifier can be used for debugging multi-process message flows. The
second part is a local timestamp. The third is an optional tag, which
you could provide with <code>seneca({tag:'foo'})</code>, although we
don't use tags in this example.

There are three _INFO_ level entries of interest. On the server-side,
the listen facility logs the fact that it has opened a TCP port, and
is now listening for connections. Then the client-side logs that is
has opened a connection to the server. And finally the server logs the
same thing.

As with the HTTP transport example above, you can split this code into
two processes by separating the client and server code. Here's the server:

```js
function color() {
  this.add( 'color:red', function(args,done){
    done(null, {hex:'#FF0000'});
  })
}

var seneca = require('seneca')

seneca()
  .use(color)
  .listen({type:'tcp'})
```

And here's the client:

```js
seneca()
  .client({type:'tcp'})
  .act('color:red')
```

HTTP and TCP are not the only transport mechanisms available. Of
course, in true Seneca-style, the other mechanisms are available as
plugins. Here's the list.

   * [redis-transport](https://github.com/rjrodger/seneca-redis-transport): uses redis for a pub-sub message distribution model
   * [beanstalk-transport](https://github.com/rjrodger/seneca-beanstalk-transport): uses beanstalkd for a message queue

If you're written your own transport plugin (see below for
instructions), and want to have it listed here, please submit a pull
request.


## Multiple Channels

... coming soon ...

## Writing Your Own Transport

... coming soon ...

<!-- message wrapper format -->


## Action Patterns

### role:transport, cmd:listen

Starts listening for actions. The <i>type</i> argument specifies the
transport mechanism. Current built-ins are <i>direct</i> (which is
HTTP), and <i>pubsub</i> (which is Redis).


### role:transport, cmd:client

Create a Seneca instance that sends actions to a remote service.  The
<i>type</i> argument specifies the transport mechanism.


## Hook Patterns

These patterns are called by the primary action patterns. Add your own for additional transport mechanisms. For example, [seneca-redis-transport](http://github.com/rjrodger/seneca-redis-transport) defines:

   * role:transport, hook:listen, type:redis
   * role:transport, hook:client, type:redis

These all take additional configuration arguments, which are passed through from the primary actions:

   * host
   * port
   * path
   * any other configuration you need

`role:transport,hook:client` should call the callback with an error, if any
occured, and an object with `match` and `send` functions. `match` should
return `true` if it can handle passed action and `false` otherwise. `send`
should call the remote side with its passed parameters and call the callback
when response is received.

```js
seneca.add('role:transport,hook:client,type:mytransport', function (args, done) {
  var myRPC = new MyRPC()
  done(null, {
    send: function(args, done) {
      myRPC.callRemoteMethod(args.foo, args, done)
    },
    match: function(args) {
      return myRPC.has(args)
    }
  })
})
```

## Pattern Selection

If you only want to transport certain action patterns, use the <i>pin</i> argument to pick these out. See the
<i>test/client-pubsub-foo.js</i> and <i>test/service-pubsub-foo.js</i> files for an example.



## Logging

To see what this plugin is doing, try:

```sh
node your-app.js --seneca.log=plugin:transport
```

To skip the action logs, use:

```sh
node your-app.js --seneca.log=type:plugin,plugin:transport
```

For more on logging, see the [seneca logging example](http://senecajs.org/logging-example.html).


## Test

This module itself does not contain any direct reference to seneca, as
it is a seneca dependency. However, seneca is needed to test it, so
the test script will perform an _npm install seneca_ (if needed). This is not
saved to _package.json_ however.

```sh
npm test
```



