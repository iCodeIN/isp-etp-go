# isp-etp-go

[![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/integration-system/isp-etp-go?color=6b9ded&sort=semver)](https://github.com/integration-system/isp-etp-go/releases)
[![GoDoc](https://godoc.org/github.com/integration-system/isp-etp-go?status.svg)](https://godoc.org/github.com/integration-system/isp-etp-go)

Client and server implementation of event transport protocol.
Websocket framework based on library [github.com/nhooyr/websocket](https://github.com/nhooyr/websocket).

## Install

```bash
go get github.com/integration-system/isp-etp-go
```

## Features:
- Rooms, broadcasting
- Store data object for connection
- [Javascript client](https://github.com/integration-system/isp-etp-js-client)
- Concurrent message write
- Context based requests

## Server example
```go
package main

import (
	"context"
	"errors"
	"github.com/integration-system/isp-etp-go"
	"log"
	"net/http"
	"nhooyr.io/websocket"
)

func main() {
	bindAddress := "127.0.0.1:7777"
	testEvent := "test_event"
	helloEvent := "hello"
	config := etp.ServerConfig{
		InsecureSkipVerify: true,
	}
	eventsServer := etp.NewServer(context.TODO(), config).
		OnConnect(func(conn etp.Conn) {
			log.Println("OnConnect", conn.ID())
		}).
		OnDisconnect(func(conn etp.Conn, err error) {
			log.Println("OnDisconnect id", conn.ID())
			log.Println("OnDisconnect err:", err)
		}).
		OnError(func(conn etp.Conn, err error) {
			log.Println("OnError err:", err)
			var closeErr websocket.CloseError
			if errors.As(err, &closeErr) {
				log.Println(closeErr.Code)
			}
			if conn != nil {
				log.Println("OnError conn ID:", conn.ID())
			}
		}).
		On(testEvent, func(conn etp.Conn, data []byte) {
			log.Println("Received " + testEvent + ":" + string(data))
		}).
		On(helloEvent, func(conn etp.Conn, data []byte) {
			log.Println("Received " + helloEvent + ":" + string(data))
			answer := "hello, " + conn.ID()
			err := conn.Emit(context.TODO(), testEvent, []byte(answer))
			log.Println("hello answer err:", err)
		})

	mux := http.NewServeMux()
	mux.HandleFunc("/isp-etp/", eventsServer.ServeHttp)
	httpServer := &http.Server{Addr: bindAddress, Handler: mux}
	go func() {
		if err := httpServer.ListenAndServe(); err != nil {
			log.Fatalf("Unable to start http server. err: %v", err)
		}
	}()

	closeCh := make(chan struct{})
	<-closeCh
}
```

## Client example
```go

package main

import (
	"context"
	"errors"
	etpclient "github.com/integration-system/isp-etp-go/client"
	"log"
	"net/http"
	"nhooyr.io/websocket"
)

func main() {
	address := "ws://127.0.0.1:7777/isp-etp/"
	testEvent := "test_event"
	helloEvent := "hello"
	config := etpclient.Config{
		HttpClient: http.DefaultClient,
	}
	client := etpclient.NewClient(config).
		OnConnect(func() {
			log.Println("Connected")
		}).
		OnDisconnect(func(err error) {
			log.Println("OnDisconnect err:", err)
		}).
		OnError(func(err error) {
			log.Println("OnError err:", err)
			var closeErr websocket.CloseError
			if errors.As(err, &closeErr) {
				log.Println(closeErr.Code)
			}
		})
	client.On(testEvent, func(data []byte) {
		log.Printf("Received %s:%s\n", testEvent, string(data))
	})
	err := client.Dial(context.TODO(), address)
	if err != nil {
		log.Fatalln("dial error:", err)
	}

	err = client.Emit(context.TODO(), helloEvent, []byte("hello"))
	if err != nil {
		log.Fatalln("emit error:", err)
	}

	closeCh := make(chan struct{})
	<-closeCh
}

```

