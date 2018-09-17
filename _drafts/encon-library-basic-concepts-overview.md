---
layout: post
title:  Encon library basic concepts overview
comments: true
---

The [encon](https://github.com/appulse-projects/encon-java/tree/master/encon) library provides a set of tools for communication with Erlang processes. It can also be used for communication with other Java processes using the same library, as well as C processes using the special library.

Since this package provides a mechanism for communicating with Erlang, message recipients can be Erlang processes or instances of `io.appulse.encon.mailbox.Mailbox`, both of which are identified with pids and possibly registered names. When pids or mailboxes are mentioned as message senders or recipients in this section, it should assumed that even Erlang processes are included, unless specified otherwise. The classes in Encon support the following:

- manipulation of data represented as Erlang data types;
- conversion of data between Java and Erlang formats;
- encoding and decoding of Erlang data types for transmission or storage;
- communication between Java nodes and Erlang processes;

In the following sections, these topics are described:

- [Mapping of basic Erlang types to Java](#mapping-of-basic-erlang-types-to-java)
- [Special mapping issues](#special-mapping-issues)
- [Nodes](#nodes)
- [Mailboxes](#mailboxes)
- [Connections](#connections)
- [Sending and receiving messages](#sending-and-receiving-messages)
- [Sending arbitrary data](#sending-arbitrary-data)
- [Linking to remote processes](#linking-to-remote-processes)
- [Using EPMD](#using-epmd)
- [Remote procedure calls](#remote-procedure-calls)
- [Tracing](#tracing)

## Mapping of basic Erlang types to Java

This section describes the mapping of Erlang basic types to Java.

| Erlang type          | Java type                                                                                                                                                |
|:---------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------|
| atom                 | [ErlangAtom](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangAtom.java)           |
| binary               | [ErlangBinary](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangBinary.java)       |
| floating point types | [ErlangFloat](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangFloat.java)         |
| integral types       | [ErlangInteger](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangInteger.java)     |
| list                 | [ErlangList](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangList.java)           |
| pid                  | [ErlangPid](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangPid.java)             |
| port                 | [ErlangPort](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangPort.java)           |
| ref                  | [ErlangReference](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangReference.java) |
| tuple                | [ErlangTuple](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangTuple.java)         |
| map                  | [ErlangMap](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangMap.java)             |
| term                 | [ErlangTerm](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/ErlangTerm.java)                |

## Special mapping issues

The atoms `true` and `false` are special atoms, used as boolean values. The class [ErlangAtom](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangAtom.java#L138) can be used to represent these.

Lists in Erlang are also used to describe sequences of printable characters (strings). A convenience class [ErlangString](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangString.java) is provided to represent Erlang strings.

## Nodes

A node as defined by Erlang/OTP is an instance of the Erlang Runtime System, a virtual machine roughly equivalent to a JVM. Each node has a unique name in the form of an identifier composed partly of the hostname on which the node is running, e.g "popa@appulse.io". Several such nodes can run on the same host as long as their names are unique. The class [Node](https://github.com/appulse-projects/encon-java/blob/master/encon/src/main/java/io/appulse/encon/Node.java) represents an Erlang node. It is created with a name and optionally a port number on which it listens for incoming connections. Before creating an instance of `Node`, ensure that `Epmd` is running on the host machine. You can you Appulse's Java implementation of [Epmd](https://github.com/appulse-projects/epmd-java/blob/master/server/README.md) server. See the Erlang documentation for more information about `Epmd`.

In this example, the host name is appended automatically to the identifier, and the port number is chosen by the underlying system:

```java
import io.appulse.encon.Node;
import io.appulse.encon.Nodes;


Node node = Nodes.singleNode("popa");
```

## Mailboxes

Erlang processes running on an Erlang node are identified by process identifiers (pids) and, optionally, by registered names unique within the node. Each Erlang process has an implicit mailbox that is used to receive messages; the mailbox is identified with the pid of the process.

Encon provides a similar mechanism with the class [Mailbox](https://github.com/appulse-projects/encon-java/blob/master/encon/src/main/java/io/appulse/encon/mailbox/Mailbox.java), a mailbox that can be used to send and receive messages asynchronously. Each `Mailbox` is identified with a unique pid and , optionally, a registered name unique within the [Node](https://github.com/appulse-projects/encon-java/blob/master/encon/src/main/java/io/appulse/encon/Node.java).

Applications are free to create mailboxes as necessary. This is done as follows:

```java
import io.appulse.encon.mailbox.Mailbox;


Mailbox mailbox = node.mailbox()
    .build();
```

The mailbox created in the above example has no registered name, although it does have a pid. The pid can be obtained from the mailbox and included in messages sent from the mailbox, so that remote processes are able to respond.

An application can register a name for a mailbox, either when the mailbox is initially created:

```java
import io.appulse.encon.mailbox.Mailbox;


Mailbox mailbox = node.mailbox()
    .name("my_process_name") // this is an optional
    .build();
```

or later on, as necessary:

```java
import io.appulse.encon.mailbox.Mailbox;


Mailbox mailbox = node.mailbox().build();
node.register(mailbox, "my_process_name");
```

Registered names are usually necessary in order to start communication, since it is impossible to know in advance the pid of a remote process. If a well-known name for one of the processes is chosen in advance and known by all communicating parties within an application, each mailbox can send an initial message to the named mailbox, which then can identify the sender pid.

## Connections

It is not necessary to explicitly set up communication with a remote node. Simply sending a message to a mailbox on that node will cause the `Node` to create a connection if one does not already exist. Once the connection is established, subsequent messages to the same node will reuse the same connection.

It is possible to check for the existence of a remote node before attempting to communicate with it. Here we send a ping message to the remote node to see if it is alive and accepting connections:

```java
if (node.ping("remote")) {
  System.out.println("remote is up");
} else {
  System.out.println("remote is not up");
}
```

If the call to `ping()` succeeds, a connection to the remote node has been established. Note that it is not necessary to ping remote nodes before communicating with them, but by using ping you can determine if the remote exists before attempting to communicate with it.

Connections are only permitted by nodes using the same security cookie. The cookie is a short string provided either as an argument when creating `Node` objects, or found in the user's home directory in the file `.erlang.cookie`. When a connection attempt is made, the string is used as part of the authentication process. If you are having trouble getting communication to work, use the trace facility (described later in this document) to show the connection establishment. A likely problem is that the cookies are different.

Connections are never broken explicitly. If a node fails or is closed, a connection may be broken however.

## Sending and receiving messages

Messages sent with this package must be instances of [ErlangTerm](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/ErlangTerm.java) or one of its subclasses. Message can be sent to processes or pids, either by specifying the pid of the remote, or its registered name and node.

In this example, we create a message containing our own pid so the echo process can reply:

```java
import static io.appulse.encon.terms.Erlang.atom;
import static io.appulse.encon.terms.Erlang.tuple;

import io.appulse.encon.terms.ErlangTerm;


ErlangTerm message = atom("hello, world");
ErlangTerm tuple = tuple(mailbox.getPid(), message);
```

When we send the message, a connection will be created:

```java
mailbox.send("popa@appulse.io", "echo", tuple);
```

And here we receive the reply:

```java
import io.appulse.encon.connection.regular.Message;
import io.appulse.encon.terms.ErlangTerm;


Message message = mailbox.receive();
ErlangTerm body = message.getBody();
```

Messages are sent asynchronously, so the call to `send()` returns as soon as the message has been dispatched to the underlying communication layer. This means that you receive no indication whether the operation completed successfully or the remote even existed. If you need this kind of confirmation, you should wait for a response from the remote process.

The echo server itself might look like this:

```java
import static io.appulse.encon.terms.Erlang.NIL;
import static io.appulse.encon.terms.Erlang.tuple;

import io.appulse.encon.mailbox.Mailbox;
import io.appulse.encon.Node;
import io.appulse.encon.Nodes;
import io.appulse.encon.terms.ErlangTerm;
import io.appulse.encon.terms.ErlangTerm;
import io.appulse.encon.terms.type.ErlangPid;


Node node = Nodes.singleNode("popa");
Mailbox mailbox = node.mailbox()
    .name("echo")
    .build();

while (true) {
  ErlangTerm message = mailbox.receive().getBody();

  ErlangPid from = message.get(0)
        .filter(ErlangTerm::isPid)
        .map(ErlangTerm::asPid)
        .orElseThrow(() -> new RuntimeException("0 element must be a PID"));

  ErlangTerm response = tuple(mailbox.getPid(), message.get(1).orElse(NIL));

  mailbox.send(from, response);
}
```

In the examples above, only one mailbox was created on each node. however you are free to create as many mailboxes on each node as you like. You are also free to create as many nodes as you like on each JVM, however because each node uses some limited system resources such as file descriptors, it is recommended that you create only a small number of nodes (such as one) on each JVM.

## Sending arbitrary data

This library was originally intended to be used for communicating between Java and Erlang, and for that reason the send and receive methods all use Java representations of Erlang data types.

However it is possible to use the library to communicate with remote processes written in Java as well, and in these cases it may be desirable to send other data types.

The simplest way to do this is to encapsulate arbitrary data in messages of type [ErlangBinary](https://github.com/appulse-projects/encon-java/blob/master/encon-terms/src/main/java/io/appulse/encon/terms/type/ErlangBinary.java). The `ErlangBinary` instance can be created from arbitrary Java objects that implement the `Serializable` or `Externalizable` interface:

```java
import static io.appulse.encon.terms.Erlang.binary;


MyClass object = new MyClass();
ErlangBinary term = binary(object);
mailbox.send(remote, term);
```

The example above will cause the object to be serialized and encapsulated in an `ErlangBinary` before being sent. The recipient will receive an `ErlangBinary` but can extract the original object from it:

```java
import static io.appulse.utils.SerializationUtils.deserialize;

import io.appulse.encon.terms.ErlangTerm;


ErlangTerm term = mailbox.receive().getBody();
if (term.isBinary()) {
  byte[] bytes = term.asBinary();
  MyClass object = (MyClass) deserialize(byte);
}
```

If you want to serialize/deserialize your `POJO`s not only between Java processes, you should to use [encon-databind](https://appulse.io/pinned/advanced-usage.html#databinding) extension.

## Linking to remote processes

Erlang defines a concept known as linked processes. A link is an implicit connection between two processes that causes an exception to be raised in one of the processes if the other process terminates for any reason. Links are bidirectional: it does not matter which of the two processes created the link or which of the linked processes eventually terminates; an exception will be raised in the remaining process. Links are also idempotent: at most one link can exist between two given processes, only one operation is necessary to remove the link.

Encon provides a similar mechanism. Also here, no distinction is made between mailboxes and Erlang processes. A link can be created to a remote mailbox or process when its pid is known:

```java
mailbox.link(remotePid);
```

The link can be removed by either of the processes in a similar manner:

```java
mailbox.unlink(remotePid);
```

If the remote process terminates while the link is still in place, an exception will be raised on a subsequent call to receive():

```java
import io.appulse.encon.mailbox.exception.ReceivedExitException;


try {
  message = mailbox.receive();
} catch (ReceivedExitException ex) {
  log.error("Remote pid {} has terminated with reason '{}'", ex.getFrom(), ex.getReason());
}
```

When a mailbox is explicitly closed, exit messages will be sent in order to break any outstanding links.

## Using EPMD

Epmd is the Erlang Port Mapper Daemon. Distributed Erlang nodes register with epmd on the localhost to indicate to other nodes that they exist and can accept connections. Epmd maintains a register of node and port number information, and when a node wishes to connect to another node, it first contacts epmd in order to find out the correct port number to connect to.

The basic interaction with EPMD is done through instances of [EpmdClient](https://github.com/appulse-projects/epmd-java/blob/master/client/src/main/java/io/appulse/epmd/java/client/EpmdClient.java) class. Nodes wishing to contact other nodes must first request information from Epmd before a connection can be set up, however this is done automatically by `Node.connect*` methods family when necessary.

When you use `Node.connect*` to connect to an Erlang node, a connection is first made to epmd and, if the node is known, a connection is then made to the Erlang node.

Java nodes can also register themselves with epmd if they want other nodes in the system to be able to find and connect to them. This is done by call to method `EpmdClient.register`.

Be aware that on some systems (such as VxWorks), a failed node will not be detected by this mechanism since the operating system does not automatically close descriptors that were left open when the node failed. If a node has failed in this way, epmd will prevent you from registering a new node with the old name, since it thinks that the old name is still in use. In this case, you must unregister the name explicitly, by using `EpmdClient.stop()`.

This will cause epmd to close the connection from the far end. Note that if the name was in fact still in use by a node, the results of this operation are unpredictable. Also, doing this does not cause the local end of the connection to close, so resources may be consumed.

## Remote procedure calls

An Erlang node acting as a client to another Erlang node typically sends a request and waits for a reply. Such a request is included in a function call at a remote node and is called a remote procedure call.

```java
import io.appulse.encon.terms.ErlangTerm;


mailbox.call("remote@appulse.io", "erlang", "date");
ErlangTerm response = mailbox.receiveRemoteProcedureResult();
```

`erlang:date/0` is just called to get the date tuple from a remote host.

> **IMPORTANT:** Note that this method has unpredicatble results if the remote node is not an `Erlang`/`Elixir` node. If you want to call remote procedures of another `Java` node, you need to create **rex** mailbox by yourself.

## Tracing

Communication between nodes can be traced by setting an `Encon` and `Netty` log levels, like this:

**src/main/resources/logback.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <include resource="io/appulse/logging/logback/base.xml"/>

  <logger name="io.appulse.encon.connection" level="DEBUG"/>
  <logger name="io.netty.handler.logging" level="DEBUG"/>
</configuration>
```
