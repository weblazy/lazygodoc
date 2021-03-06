# Tcp

```
package tcp_protocol

import (
	"fmt"
	"net"

	"github.com/weblazy/socket-cluster/logx"
	"github.com/weblazy/socket-cluster/protocol"
)

type TcpProtocol struct {
}

func (this *TcpProtocol) ListenAndServe(port int64, onConnect func(conn protocol.Connection)) error {
	listener, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
	if err != nil {
		return err
	}
	go func() {
		for {
			connect, err := listener.Accept()
			if err != nil {
				logx.LogHandler.Error(err)
				break
			}
			go func(connect net.Conn) {
				conn := NewTcpConnection(connect)
				onConnect(conn)
			}(connect)
		}
	}()
	return nil
}

func (this *TcpProtocol) Dial(addr string) (protocol.Connection, error) {
	tcpAddr, err := net.ResolveTCPAddr("tcp4", addr)
	if err != nil {
		return nil, err
	}
	conn, err := net.DialTCP("tcp", nil, tcpAddr)
	if err != nil {
		return nil, err
	}
	return NewTcpConnection(conn), err
}
```

```
package tcp_protocol

import (
	"net"
	"sync"

	"github.com/weblazy/socket-cluster/protocol"
)

type TcpConnection struct {
	Conn                net.Conn
	Mutex               sync.Mutex
	flowProtocolHandler protocol.Proto
}

func NewTcpConnection(conn net.Conn) *TcpConnection {
	return &TcpConnection{
		Conn:                conn,
		flowProtocolHandler: protocol.NewFlowProtocol(protocol.HEADER, protocol.MAX_LENGTH, conn),
	}
}

// WriteMsg send byte array message
func (this *TcpConnection) WriteMsg(data []byte) error {
	data, err := this.flowProtocolHandler.Pack(data)
	if err != nil {
		return err
	}
	this.Mutex.Lock()
	defer this.Mutex.Unlock()
	_, err = this.Conn.Write(data)
	return err
}

func (this *TcpConnection) ReadMsg() ([]byte, error) {
	msg, err := this.flowProtocolHandler.ReadMsg()
	if err != nil {
		return nil, err
	}
	return msg, err
}

func (this *TcpConnection) Addr() string {
	return this.Conn.RemoteAddr().String()
}

func (this *TcpConnection) Close() error {
	return this.Conn.Close()
}

func (this *TcpConnection) SetFlowProtocolHandler(flowProtocolHandler protocol.Proto) {
	this.flowProtocolHandler = flowProtocolHandler
}

```