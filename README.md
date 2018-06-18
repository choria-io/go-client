# Golang Choria Client

This is a low level client to the Choria network protocol.  It is not intended to be used for making `mcollective` compatible RPC calls - such a library would be written on top of it.

It's a high performance client that can make multiple parallel connections to the Choria Network Broker and multiplex data into the calling client.  It's been shown to do full `ping` style round trip to 50 000 nodes in less than 1 second.

[![GoDoc](https://godoc.org/github.com/choria-io/go-client?status.svg)](https://godoc.org/github.com/choria-io/go-client)

## Features

  * Basic client for sending any arbitrary data over the Choria network
  * Constructors for Choria filters from strings like those a CLI would get when parsing user input
  * `mcollective` like broadcast discovery client

## Basic Usage

The client can publish and receive messages via the `choria.Message` structure, here's the most basic `ping` client:

*NOTE: this is only the core code, obvious imports etc are omitted, your IDE will add them automatically*

```go
package main

import (
   	"github.com/choria-io/go-client/client"
	"github.com/choria-io/go-choria/choria"
	"github.com/choria-io/go-protocol/protocol"
	log "github.com/sirupsen/logrus"
)

func panicIf(err error) {
    if err != nil {
        panic(err)
    }
}

func main() {
    fw, err := choria.New(choria.UserConfig())
    panicIf(err)

    // discovery needs the word 'ping', we're sending it to the mcollective sub collective
    msg, err := fw.NewMessage(base64.StdEncoding.EncodeToString([]byte("ping")), "discovery", "mcollective", "request", nil)
    panicIf(err)

    // create a basic class filter matching nodes with "choria::broker" class,
    // can also use AgentFilter, IdentityFilter, CompoundFilter, FactFilter,
    // CombinedFilter in any combination
    msg.Filter, err = client.NewFilter(client.ClassFilter("choria::broker")
    panicIf(err)

    msg.SetProtocolVersion(protocol.RequestV1)
    msg.SetReplyTo(choria.ReplyTarget(msg, msg.RequestID))

    // severl options exist like Timeout, Receivers and Log
    cl, err := client.New(fw, client.Timeout(20))
    panicIf(err)

    ctx, cancel = context.WithCancel(context.Backgroun())
    defer cancel()

    // publishes the request in msg and calls handler for each, when Receivers() are more than
    // 1 - 3 is the default - this will be called multiple times concurrently, use a mutex if you
    // must or set Receivers(1) when creating the client
    cl.Request(ctx, msg, handler)
    panicIf(err)
}

func handler(ctx context.Context, m *choria.ConnectorMessage) {
    reply, err := b.fw.NewTransportFromJSON(string(m.Data))
    if err != nil {
        log.Errorf("Could not process a reply: %s", err)
        return
    }

    fmt.Println(reply.SenderID())
}
```

## Broadcast Discovery

The above code is basically a discovery, there's a more fleshed out and simpler way to go about it like this:

```go
import (
   	"github.com/choria-io/go-client/broadcast"
    "github.com/choria-io/go-choria/choria"
	"github.com/choria-io/go-protocol/protocol"
)

func main() {
    fw, err := choria.New(choria.UserConfig())
    panicIf(err)

    b := broadcast.New(fw)

    nodes, err := b.Discover(ctx, broadcast.Filter(protocol.NewFilter()), broadcast.Timeout(10*time.Second), broadcast.Collective("test"))

    fmt.Printf("Discovered %d nodes", len(nodes))
}
```