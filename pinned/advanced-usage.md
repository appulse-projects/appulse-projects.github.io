---
layout: page
title:  Advanced Usage
---

- [Databinding](#databinding)
  - [Databind dependency](#databind-dependency)
  - [Pojo declaration](#pojo-declaration)
  - [Serialization](#serialization)
  - [Deserialization](#deserialization)
- [Handler](#handler)
  - [Handler dependency](#handler-dependency)
  - [Basic handler](#basic-handler)
  - [Advanced handler](#advanced-handler)
- [Spring](#spring)
  - [Spring dependency](#spring-dependency)
  - [Configuration](#configuration)
  - [Server](#server)
  - [Client](#client)

## Databinding

### Databind dependency

Adding databind's dependency to your `JVM` app:

**Maven**:

```xml
<dependencies>
  ...
  <dependency>
    <groupId>io.appulse.encon</groupId>
    <artifactId>encon</artifactId>
    <version>1.6.7</version>
  </dependency>
  <dependency>
    <groupId>io.appulse.encon</groupId>
    <artifactId>encon-databind</artifactId>
    <version>1.6.7</version>
  </dependency>
  ...
</dependencies>
```

**Gradle**:

```groovy
dependencies {
  compile 'io.appulse.encon:encon:1.6.7'
  compile 'io.appulse.encon:encon-databind:1.6.7'
}
```

### Pojo declaration

Let's imagine, you have POJO like this:

```java
import java.util.List;
import java.util.Set;

import io.appulse.encon.databind.annotation.AsErlangAtom;
import io.appulse.encon.databind.annotation.AsErlangList;
import io.appulse.encon.databind.annotation.IgnoreField;


// Serialize/deserialize object to/from Erlang's list
@AsErlangList
public class Pojo {

  // it will be serialized as ErlangString
  String name;

  // it will be serialized as ErlangInteger
  int age;

  // it will be serialized as ErlangAtom
  boolean male;

  // this field will be ignored
  @IgnoreField
  int ignored;

  // it will be serialized as ErlangList with ErlangString elements
  List<String> languages;

  // this field will be serialized as ErlangAtom, obviously
  @AsErlangAtom
  String position;

  // it will be serialized as ErlangList with ErlangString elements
  @AsErlangList
  Set<String> set;

  // this string will be serialized as ErlangList with ErlangInteger elements
  @AsErlangList
  String listString;

  // it will be serialized as ErlangList with ErlangAtom elements
  @AsErlangList
  Boolean[] bools;
}
```

### Serialization

```java
import io.appulse.encon.databind.TermMapper;
import io.appulse.encon.terms.ErlangTerm;


Pojo pojo = // ...
ErlangTerm term = TermMapper.serialize(pojo);
```

### Deserialization

```java
import io.appulse.encon.databind.TermMapper;
import io.appulse.encon.terms.ErlangTerm;


Pojo result = TermMapper.deserialize(term, Pojo.class);
```

## Handler

It is very boring to write every time a routine code like [this](https://appulse.io/pinned/get-started.html#receive-message-in-java) for receiving the messages, so you can write a business logic handler and start it in separate thread with help of `encon-handler` dependency.

### Handler dependency

Adding handler's dependency to your `JVM` app:

**Maven**:

```xml
<dependencies>
  ...
  <dependency>
    <groupId>io.appulse.encon</groupId>
    <artifactId>encon-handler</artifactId>
    <version>1.6.7</version>
  </dependency>
  ...
</dependencies>
```

**Gradle**:

```groovy
dependencies {
  compile 'io.appulse.encon:encon-handler:1.6.7'
}
```

### Basic handler

The simplest way to create your own handler is - implement a `MessageHandler` interface:

```java
import io.appulse.encon.connection.control.ControlMessage;
import io.appulse.encon.handler.message.MessageHandler;
import io.appulse.encon.mailbox.Mailbox;
import io.appulse.encon.terms.ErlangTerm;


class DummyMessageHandler implements MessageHandler {

  @Override
  public void handle (Mailbox self, ControlMessage header, ErlangTerm body) {
    System.out.println("A new message!");
  }
}
```

Now, we can start our handler with the following code:

```java
import io.appulse.encon.handler.mailbox.BlockingMailboxHandler;
import io.appulse.encon.handler.mailbox.MailboxHandler;


MailboxHandler handler = BlockingMailboxHandler.builder()
  .messageHandler(new DummyMessageHandler())
  .mailbox(myMailbox)
  .build();

// start it in a separate thread:
handler.startExecutor();
// ...
handler.close();
```

## Advanced handler

Let's imagine we already have, for example, two services:

```java
public static class MyService1 {

  public void handler1 () {
    // ...
  }

  public void handler2 (int a, String b, boolean c) {
    // ...
  }
}

public static class MyService2 {

  public void popa (int a) throws Exception {
    // ...
  }
}
```

And now, we want to use them for handling our incoming messages. We can do it with the following code:

```java
import static io.appulse.encon.handler.message.matcher.Matchers.anyString;
import static io.appulse.encon.handler.message.matcher.Matchers.anyInt;
import static io.appulse.encon.handler.message.matcher.Matchers.eq;

import io.appulse.encon.handler.mailbox.DefaultMailboxHandler;
import io.appulse.encon.handler.mailbox.MailboxHandler;
import io.appulse.encon.handler.message.MessageHandler;
import io.appulse.encon.handler.message.matcher.MethodMatcherMessageHandler;


MyService1 service1 = new MyService1();
MyService2 service2 = new MyService2();

MessageHandler messageHandler = MethodMatcherMessageHandler.builder()
    .wrap(service1)
        // redirects '[]' (empty list) to method MyService1.handler1
        .list(it -> it.handler1())
        // redirects tuple {any_number, any_string, atom(true)}
        // to MyService1.handler2
        .tuple(it -> it.handler2(anyInt(), anyString(), eq(true)))
    .wrap(service2)
        // redirects {42} to MyService2.popa
        .none(it -> it.handler3(42))
    .build();

MailboxHandler mailboxHandler = DefaultMailboxHandler.builder()
    .messageHandler(messageHandler)
    .mailbox(myMailbox)
    .build();

// start it in a separate thread:
mailboxHandler.startExecutor();

// ...

// handler is a Closeable object
mailboxHandler.close();
```

## Spring

### Spring dependency

Adding Spring's dependency to your `JVM` app:

**Maven**:

```xml
<dependencies>
  ...
  <dependency>
    <groupId>io.appulse.encon</groupId>
    <artifactId>encon-spring</artifactId>
    <version>1.6.7</version>
  </dependency>
  ...
</dependencies>
```

**Gradle**:

```groovy
compile 'io.appulse.encon:encon-spring:1.6.7'
```

### Configuration

The dependency contains automatic [configuration](https://github.com/appulse-projects/encon-java/blob/master/encon-spring/src/main/java/io/appulse/encon/spring/EnconAutoConfiguration.java), so you just need to tune only your `YAML`/`properties` configs.

The `application.yml` file could looks like:

```yaml
spring.encon:
  enabled: true

  # The same object as io.appulse.encon.config.Defaults
  defaults:
    epmd-port: 8888
    type: R3_ERLANG
    short-name: true
    cookie: secret
    protocol: TCP
    low-version: R4
    high-version: R6
    distribution-flags:
      - MAP_TAG
      - BIG_CREATION
    server:
      boss-threads: 2
      worker-threads: 4

  # Map of node_name -> io.appulse.encon.config.NodeConfig
  # the missing configurations take from 'defaults' block
  nodes:
    node-1:
      epmd-port: 7373
      type: R3_HIDDEN
      cookie: non-secret
      protocol: SCTP
      low-version: R5C
      high-version: R6
      distribution-flags:
        - EXTENDED_REFERENCES
        - EXTENDED_PIDS_PORTS
        - BIT_BINARIES
        - NEW_FLOATS
        - FUN_TAGS
        - NEW_FUN_TAGS
        - UTF8_ATOMS
        - MAP_TAG
        - BIG_CREATION
      mailboxes:
        - name: net_kernel
        - name: another
        - name: another_one
      server:
        port: 8971
        boss-threads: 1
        worker-threads: 2

    node-2:
      short-name: false
      cookie: popa
      mailboxes:
        - name: net_kernel
```

### Server

> **NOTICE:** `@ErlangMailbox` annotation creates an application context component, so it could be `autowired`/`injected`.

> **NOTICE:** `nodes` and `mailboxes` declared in `@ErlangMailbox` annotation, but not existed in the application context, will be created and registered automatically.

```java
import static lombok.AccessLevel.PRIVATE;

import java.util.List;
import java.util.Set;

import io.appulse.encon.databind.annotation.AsErlangAtom;
import io.appulse.encon.databind.annotation.AsErlangList;
import io.appulse.encon.databind.annotation.AsErlangTuple;
import io.appulse.encon.databind.annotation.IgnoreField;
import io.appulse.encon.spring.ErlangMailbox;
import io.appulse.encon.spring.InjectMailbox;
import io.appulse.encon.spring.MailboxOperations;
import io.appulse.encon.spring.MatchingCaseMapping;
import io.appulse.encon.terms.ErlangTerm;
import io.appulse.encon.terms.type.ErlangInteger;
import io.appulse.encon.terms.type.ErlangPid;

import lombok.Data;
import lombok.EqualsAndHashCode;
import lombok.experimental.Delegate;
import lombok.experimental.FieldDefaults;
import lombok.NoArgsConstructor;


@ErlangMailbox(           // registers a new Bean
    node = "echo-server", // node's name, it will be autocreated
    name = "echo"         // mailbox's name for "self" field (see it below)
)
class EchoServer {

  @Delegate // use Lombok's helper annotation, for easy mailbox use
  @InjectMailbox // instead of @Autowire/@Inject you MUST use this annotation
  MailboxOperations self;

  // receives a tuple: {pid, 42, payload}
  @MatchingCaseMapping("{any(), eq(42), any()}")
  public void handle (ErlangPid sender, ErlangInteger number, ErlangTerm payload) {
    // some code

    // send response back
    self.send(sender, tuple("your", "response", "object"));
  }

  // receives a tuple with POJO's fields
  @MatchingCaseMapping
  public void handle (MyPojo request) {
    // some code
  }

  @Data
  @AsErlangTuple // the way, how to wrap POJO's fields
  @NoArgsConstructor // be sure, what you have a no-args constructor
  @FieldDefaults(level = PRIVATE)
  @EqualsAndHashCode(exclude = "ignored")
  public static class MyPojo {

    String name; // ErlangString

    int age; // ErlangInteger

    boolean male; // ErlangAtom

    // this field will be ignored
    // during serialization/deserialization
    @IgnoreField
    int ignored;

    List<String> languages; // ErlangList<ErlangString>

    @AsErlangAtom
    String position; // ErlangAtom

    @AsErlangList
    Set<String> set; // ErlangList<ErlangString>

    @AsErlangList
    String listString; // ErlangList
  }
}
```

### Client

> **NOTICE:** `@ErlangMailbox` annotation creates an application context component, so it could be `autowired`/`injected`.

> **NOTICE:** `nodes` and `mailboxes` declared in `@ErlangMailbox` annotation, but not existed in the application context, will be created and registered automatically.у

```java
@ErlangMailbox(
    node = "echo-client"
)
class EchoClient {

  @Delegate
  @InjectMailbox
  MailboxOperations self;
}
```
