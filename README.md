__Switch is a relais station__

# Motivation

Sometimes you need to call some web service from you local development
environment (your PC). The web service is located in your customer's
private LAN and you haven't setup a VPN yet. You'd like to put
everything into your favorite public cloud. But your customer has some
legacy system that you're trying to integrate and which cannot be put
there.

So you need a way to call into your customer's private LAN eventhough
there is no direct TCP/IP route.

# Enter Switch

The idea is to run a Switch server somewhere where you and your
customer can connect to it --- like some public cloud, but it could
also be your company's DMZ [1].

The Switch server will act like a proxy. Your web service client will
connect to the Switch server and send HTTP requests to it (either
using the Switch server as a proxy or as a target) and wait for a
response (synchronuous call).

The Switch server accepts/reads those requests. Since the Switch
server cannot call/forward the request into your customer's LAN, it
waits for someone to ask for those requests --- i.e. someone has to
pull those requests out of the Switch server.

Therefore there is another Switch server ("polling server") which runs
in your customer's LAN. This Switch server connects to the first
Switch server ("passive server") and polls/asks for requests.

Once the polling Switch server has received the request data, it
forwards it to the target web service.

The polling Switch server then transports the target server's response
back by pushing it to the passive Switch server which in turn will use
it as the response to the request of your web service client.

[1] https://en.wikipedia.org/wiki/DMZ_(computing)

# Switch structure

There are four parts ("nodes") that make up Switch:

__S-node__ (runs in _passive server_): An S-node is a ring handler
function. It is called by the ring servlet in response to a client's
HTTP request. The S-node puts the request data onto a `core.async`
channel ("request channel") and then waits for data to arrive on
another `core.asyn` "response channel".

__P-node__ (runs in _passive server_): A P-node is a ring handler
function. It is called by the ring servlet in response to the polling
server's HTTP request (see Q-node below). The P-node waits for data on
a `core.async` "request channel" and then returns that.

__Q-node__ (runs in _polling server_): A Q-node actively makes HTTP
calls to the P-node (looping). The Q-node will call the P-node in
order to poll requests. And the Q-node will call the P-node in order
to push target server's responses. Note that both of these may or may
not take place in just one single call. So it may well be that there
is more than one "active" HTTP call from the Q-node to the P-node at a
time. When the Q-node receives request data from the P-node it will
put that data on a `core.sync` request channel and then loop and ask
the P-node for more requests. The Q-node will also (concurrently) wait
for data from the C-node (see below) to arrive on the `core.async`
"response channel". When the data arrives the Q-node will call the
P-node and push the response data.

__C-node__ (runs in _polling server_): A C-node waits for data on a
`core.async` "request channel". When the data arrives the C-node
synchronuously calls the target server. The C-node then puts the
target server's response on a `core.async` "response channel".

Recap:

* S-nodes play the role of a (passive) server. They offer the web
  service endpoint and delegate the processing to some (async) backend
  (P-nodes).

* C-nodes play the role of a web service client. They are driven by
  data which arrives on a "request channel" and put target server's
  responses on a response channel.

* P-nodes do two things: they consume data from a "request channel"
  and hold ("buffer") this data until the Q-node asks for it (note
  that the channel itself may be the buffer). And they process
  requests from Q-nodes by (a) putting response data contained in the
  request on a "response channel" and (b) by pulling data from the
  "request channel" (buffer) in order to return this data to the
  Q-node's request. Note that all this can be done in just one ring
  handler function.

* Q-nodes also do two things: they wait for data (from C-node) on a
  "response channel" and push this data back by calling the
  P-node. And they call ("poll") the P-node for request data. Note
  again that all this can be done with just one loop.

* By connecting S-node to a C-node you have a simple proxy that just
  routes request data back and forth.

* You can use P-nodes and Q-nodes to connect `core.async` data
  sources/sinks over a network. So they act like JMS
  broker/client. That said P-nodes and Q-nodes will be implemented as
  queue broker and client. Some links:
  
  * https://github.com/ruedigergad/bowerick
  
  * https://ruedigergad.com/2016/08/07/bowerick-easing-simple-message-oriented-middleware-tasks-with-clojure-and-java/

  * https://github.com/lupapiste/jms-client

  * https://github.com/sbtourist/clamq

  * https://www.predic8.com/activemq-hornetq-rabbitmq-apollo-qpid-comparison.htm

  * "Message Queues" in https://www.clojure-toolbox.com/
