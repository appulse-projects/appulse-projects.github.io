---
layout: page
title:  Examples
---

Talk is cheap! Show me some code!

- [Receive and send back](#receive-and-send-back)

## Receive and send back

```java
import static io.appulse.encon.terms.Erlang.string;

import io.appulse.encon.Node;
import io.appulse.encon.Nodes;
import io.appulse.encon.config.NodeConfig;
import io.appulse.encon.mailbox.Mailbox;
import io.appulse.encon.terms.ErlangTerm;
import io.appulse.encon.terms.type.ErlangPid;


public class Main {

  public static void main(String[] args) {
    NodeConfig config = NodeConfig.builder()
        .shortNamed(true)
        .build();

    Node node = Nodes.singleNode("java@localhost", config);

    System.out.println("node descriptor: " + node.getDescriptor().toString());

    Mailbox mailbox = node.mailbox()
        .name("my_process")
        .build();

    // receives a tuple {pid, string}
    ErlangTerm payload = mailbox.receive().getBody();

    ErlangPid from = payload.get(0)
        .filter(ErlangTerm::isPid)
        .map(ErlangTerm::asPid)
        .orElseThrow(() -> new RuntimeException("Expected first element is PID"));

    String text = payload.get(1)
        .filter(ErlangTerm::isTextual)
        .map(ErlangTerm::asText)
        .orElseThrow(() -> new RuntimeException("Expected second element is string"));

    System.out.format("from %s message: %s\n", from, text);

    // sends back a string
    mailbox.send(from, string("hello world"));

    // and close the node
    node.close();
  }
}
```
