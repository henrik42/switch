# Introduction to Switch

Switch tunnels HTTP traffic between private LANs over the internet. So
Switch acts like a forward HTTP proxy. 

# Motivation

Sometimes you need to call some web service from you local development
environment (your PC, your private LAN). The web service is located in
your customer's private LAN and you haven't setup a VPN yet. You'd
like to put everything into your favorite public cloud. But your
customer has some legacy system that you're trying to integrate and
which cannot be put there.

So you need a way to call into your customer's private LAN eventhough
there is no direct TCP/IP route.

# Enter Switch

The idea is to run a Switch server somewhere where you and your
customer can connect to it --- like some public cloud, but it could
also be your company's DMZ [1].

To the requesting site the Switch server will act like a proxy. Your
web service client on your PC will connect to the Switch server and
send HTTP requests to it (either using the Switch server as a proxy or
as a target) and wait for a response (synchronuous call).

The Switch server accepts/reads those requests. Since the Switch
server cannot call/forward the request into your customer's LAN, it
waits for someone to ask for those requests --- i.e. someone has to
pull those requests out of the Switch server.

Therefore there is another Switch server (let's call it the _minor
server_) which runs in your customer's LAN. The _minor_ Switch server
connects to the first Switch server (let's call this one the _major
server_) and polls/asks for requests.

Once the _minor_ Switch server has received the request data, it
forwards it to the target web service --- i.e. it invokes the target
web service.

The _minor_ Switch server then transports the target server's response
back by pushing it to the _major_ Switch server which in turn will use
it as the response to the request of your waiting web service client.

[1] https://en.wikipedia.org/wiki/DMZ_(computing)

# Switch structure

The _major_ Switch server is a Jetty based Ring handler that processes
HTTP requests by putting them on a `core.async` _request channel_. It
then waits for data on a `core.async` _response channel_. When the
data arrives it is given back as the Ring response to the calling HTTP
client.

Data on the _request channel_ is given to an embedded ActiveMQ broker.

The _minor_ Switch server uses an ActiveMQ client to connect to the
_major_ Switch server's ActiveMQ broker (via HTTPS if you like) and
receives the request data. It then sends back the target server's
response data to the ActiveMQ broker.

[x] Jetty
[x] Ring
[x] core.async
[x] https://exupero.org/hazard/post/clojure-proxy/
[x] https://activemq.apache.org/http-and-https-transports-reference
[x] https://stackoverflow.com/questions/14480023/activemq-and-embedded-broker
[x] http://activemq.apache.org/how-do-i-embed-a-broker-inside-a-connection.html
[x] https://dataissexy.wordpress.com/2018/11/05/too-small-to-kafka-but-too-big-to-wait-really-simple-streaming-in-clojure-queues-pubsub-activemq-rabbitmq/
[x] https://github.com/sbtourist/clamq/blob/master/clamq-activemq/test/clamq/test/activemq_test.clj
[x] https://github.com/puppetlabs/mq
[x] https://ruedigergad.com/2016/08/07/bowerick-easing-simple-message-oriented-middleware-tasks-with-clojure-and-java/
