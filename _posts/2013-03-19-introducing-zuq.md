---
title: "Introducing ZuQ - A Simple ZeroMQ Queuing Daemon"
date: "2013-03-19"
tags: 
  - "c"
  - "php"
  - "queuing-system"
  - "rabbitmq"
  - "zeromq"
  - "zmq"
  - "zuq"
---

We recently had the need to create a queuing system to replace an implementation of RabbitMQ that was being used on a previous project. The reasoning behind this is that the requirements of the project required a very custom implementation of a queuing system that would drastically alter in architecture as the project grew and RabbitMQ just wasn't going to fit the bill. However to start with we required something super simple and efficient that could be expanded and developed as required. After a little investigation and a lot of recommendation from others we decided to use ZeroMQ as our transport layer for that very reason, as we could build something which could span across multiple servers and was fast.

<!--more-->

[![This is a simple Queue and Client Diagram](/assets/images/clientQueue.png)](http://phpboyscout.uk/wp-content/uploads/2013/03/clientQueue.png)

The diagram above helps describe our basic queuing system. We have a queue daemon that is continuously listening for connections and two clients, one that populates the queue and the other that retrieves from the queue.

## The Clients

Each client is written in PHP and uses a 0mq socket to communicate with a service, in this case our queue service. We used a SOCKET\_REQ type of socket in order to have a request/response communication with our queue service.

> ```
> function client_socket(\ZMQContext $context)
> {
>     // SOCKET_REQ used to create a client that sends requests to and receive from a service
>     $client = new \ZMQSocket($context,\ZMQ::SOCKET_REQ);
>     $client->connect("tcp://localhost:5555");
>     // SOCKOPT_LINGER = 0 Configure socket to not wait at close time
>     $client->setSockOpt(\ZMQ::SOCKOPT_LINGER, 0);
>     return $client;
> }
> 
> public function injectIntoQueue()
> {
>     $context = new \ZMQContext();
>     $client = $this->client_socket($context);
>     $msg = "This is a message";
>     $retries_left = 3;
>     $read = $write = array();
>     while ($retries_left) {
>         // We send a request, then we wait to get a reply
>         $client->send($msg);
>         $expect_reply = true;
>         while ($expect_reply) {
>             // Poll socket for a reply, with timeout
>             $poll = new \ZMQPoll();
>             $poll->add($client, \ZMQ::POLL_IN);
>             $events = $poll->poll($read, $write, 2500);
>             // If we got a reply, process it
>             if ($events > 0) {
>                 // We got a reply from the server, must match sequence
>                 $reply = $client->recv();
>                 if (intval($reply) == $msg) {
>                     $retries_left = 0;
>                     $expect_reply = false;
>                 }
>              } elseif (--$retries_left == 0) {
>                 break;
>              } else {
>                 // Old socket will be confused; close it and open a new one
>                 $client = $this->client_socket($context);
>                 // Send request again, on new socket
>                 $client->send($msg);
>             }
>         }
>     }
> }
> ```

You can see from the code above, have a 3 strike rule. The reasoning behind this is that if the client fails to connect to the queue service more than 3 times, we can stop trying to inject into the queue and move on to the next item. As we ultimately intend to adapt the [lazy pirate pattern](http://zguide.zeromq.org/page:all#Client-side-Reliability-Lazy-Pirate-Pattern "lazy pirate pattern") we have made it so that if the socket times out, we can then create a new socket and retry. Without this, as the architecture becomes more complicated we may then end up in a situation where we might have errors, thus the recommend solution is to create a new socket. Once the client has sent its message to the queue, we poll for a response (i.e. which is the message we sent returned back). Once we have a response that is valid, meaning that the queue has been populated, we can stop polling until the next message.

The Frontend Client

> ```
> protected function getFromQueue()
> {
>     $context = new \ZMQContext();
>     $worker = new \ZMQSocket($context, \ZMQ::SOCKET_REQ);
>     $read = $write = array();
>     // Set random identity to make tracing easier
>     $worker->connect("tcp://localhost:5556");
>     // Tell queue we're ready for work
>     $worker->send("ready");
>     $reply = $worker->recv();
>     return $reply;
> }
> ```

Our Frontend Client is much simpler as it is part of a process that is being continually updated, therefore it doesn't need the same connection retries are the Backend Client. We simply send "ready" to the queue system and if the queue is populated it will return us the first item.

## The Queuing Service

This is a continuously running executable created using c++.

> ```
> zmq::context_t context(1);
> zmq::socket_t frontend (context, ZMQ_ROUTER);
> zmq::socket_t backend (context, ZMQ_ROUTER);
> backend.bind("tcp://*:5555");
> frontend.bind("tcp://*:5556");
> ```

We create two sockets of type ZMQ\_ROUTER which is an advanced pattern used for extending request/reply sockets. This means when we improve our queuing system we will be able to route packets to specific recipients using an address in the message.

After creating our sockets, we initialise them

> ```
> // Initialize poll set
> zmq::pollitem_t items [] = {
>     { frontend, 0, ZMQ_POLLIN, 0 },
>     { backend, 0, ZMQ_POLLIN, 0 }
> };
> 
> //poll the sockets - this seems to poll both sockets at the same time
> zmq::poll (items, 2, -1);
> ```

## Backend Handler

If we get a message from the backend, we check the contents to see if it contains purge at which point we empty the queue, otherwise we push the msg contents onto the queue. Finally we send the message back to the backend to show that the message has been received.

> ```
> //receive msg from client
> if (items [1].revents & ZMQ_POLLIN) {
>     //get message from client
>     zmq::message_t message(0);
>     string client_addr = s_recv (backend);
>     string empty = s_recv (backend);
>     assert (empty.size() == 0);
>     string msg = s_recv (backend);
> 
>     //allow the backend to purge the queue
>     if(msg == "purge") {
>         while (!queue.empty()) queue.pop();
>     } else {
>         queue.push(msg);
>     }
> 
>     //send response back to the backend
>     s_sendmore (backend, client_addr);
>     s_sendmore (backend, "");
>     s_send (backend, msg);
> }
> ```
> 
>  

## Frontend Handler

If we get the "ready" message from the frontend client, we pop a message off the queue and return it to the frontend client. If the queue is empty we send an "empty" message back instead.

> ```
> // Handle activity on frontend
> if (items [0].revents & ZMQ_POLLIN) {
>     //get message from worker
>     zmq::message_t message(0);
>     string worker_addr = s_recv (frontend);
>     string empty = s_recv (frontend);
>     assert (empty.size() == 0);}
>     string msg = s_recv (frontend); string queueMsg;
>     if(msg == "ready") {
>         if(queue.size() > 0) {
>             queueMsg = queue.front();
>             queue.pop();
>         } else {
>             queueMsg = "empty";
>         }
>     }
>     //send reply to worker with contents of queue
>     s_sendmore (frontend, worker_addr);
>     s_sendmore (frontend, "");
>     s_send (frontend, queueMsg);
> }
> ```

That is our complete queuing system using ZeroMQ with PHP and C++.

## Summary

Using the above has allowed us to create a very simple in memory queue daemon that we can use to quickly pass data from one system to a another. On the whole it works well and we are looking to expand on it in the near future to increase both its functionality and scalability.

You can find the queueing daemon (christened as "ZuQ") on github @ [https://github.com/zucchi/ZuQ](https://github.com/zucchi/ZuQ)
