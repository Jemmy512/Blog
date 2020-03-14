# Event Loop

```C++
struct connection {
    ConnectionType *type;
    ConnectionState state;  // CONN_STATE_{CONNECTED, CONNECTING, CONNECT, ...}
    int flags;
    int last_errno;
    void *private_data;     // associated client pointer
    ConnectionCallbackFunc conn_handler;    // user defiend handler
    ConnectionCallbackFunc write_handler;   // user defiend handler
    ConnectionCallbackFunc read_handler;    // user defiend handler
    int fd;
};

ConnectionType CT_Socket = {
    // unified handler for readble/writable event from epoll
    .ae_handler = connSocketEventHandler,
    .close = connSocketClose,   // system call close
    .write = connSocketWrite,   // system call write
    .read = connSocketRead,     // system call read
    .accept = connSocketAccept,
    .connect = connSocketConnect,
    .set_write_handler = connSocketSetWriteHandler,
    // set user defined read handler called when epoll detects it's readable
    .set_read_handler = connSocketSetReadHandler,
    .get_last_error = connSocketGetLastError,
    .blocking_connect = connSocketBlockingConnect,
    .sync_write = connSocketSyncWrite,
    .sync_read = connSocketSyncRead,
    .sync_readline = connSocketSyncReadLine
};
```

Redis encapsulate file event to

```C++
typedef struct aeFileEvent {
    int mask;               // AE_(READABLE|WRITABLE|BARRIER)
    aeFileProc *rfileProc;   // readable file procedure
    aeFileProc *wfileProc;   // writable file procedure
    void *clientData;       // client pointer
} aeFileEvent;
```

Redis stores every registered event in the array of aeFileEvent in  aeEventLoop, the index of the array associates with the fd.

```C++
typedef struct aeEventLoop {
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
} aeEventLoop;
```

### Accept a Client Connection Request

1. When server start, register listen socket read event to epoll, and create the associated **aeFileEvent** and set its **rfileProc** to **acceptTcpHandler**.

   ```C++
   aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE, acceptTcpHandler,NULL)
   ```

2. When epoll detects the listen socket is readable, **acceptTcpHandler** accepts the request, call **connCreateAcceptedSocket** to create a **connection** object, sets its state to **CONN_STATE_ACCEPTING**

3. **acceptCommonHandler** calls **createClient** to create a redis client, sets client no delay, non-block, and connection read handler to **readQueryFromClient**, associates the connection with the client

   ```C++
   if (conn) {
     connNonBlock(conn);
     connEnableTcpNoDelay(conn);
     if (server.tcpkeepalive)
       connKeepAlive(conn,server.tcpkeepalive);
     connSetReadHandler(conn, readQueryFromClient);
     connSetPrivateData(conn, c);
   }
   ```

4. When setting the connection read handler, also register read event of the new client in epoll, set read handler to **connection->type->ae_handler(connSocketEventHandler)** which is a unified handler to handle connection, readable and writable event.

   ```C++
   aeCreateFileEvent(server.el,conn->fd, AE_READABLE,conn->type->ae_handler,conn)
   ```

5. **acceptCommonHandler** calls **connAccept**, set connection state to **CONN_STATE_CONNECTED**, and then call **clientAcceptHandler**

### Read Client Query Request

1. epoll returns if a client is readable, calls it's read handler(connSocketEventHandler) set at previous section step 4.

   ```C++
   static void connSocketEventHandler(struct aeEventLoop *el, int fd, void *clientData, int mask)
   {
     connection *conn = clientData;

        // handle connection event
       if (conn->state == CONN_STATE_CONNECTING &&
               (mask & AE_WRITABLE) && conn->conn_handler) {

           if (connGetSocketError(conn)) {
               conn->last_errno = errno;
               conn->state = CONN_STATE_ERROR;
           } else {
               conn->state = CONN_STATE_CONNECTED;
           }

           if (!conn->write_handler) aeDeleteFileEvent(server.el,conn->fd,AE_WRITABLE);

           if (!callHandler(conn, conn->conn_handler)) return;
           conn->conn_handler = NULL;
       }

       int invert = conn->flags & CONN_FLAG_WRITE_BARRIER;

       int call_write = (mask & AE_WRITABLE) && conn->write_handler;
       int call_read = (mask & AE_READABLE) && conn->read_handler;

       /* Handle normal I/O flows */
       if (!invert && call_read) {
           if (!callHandler(conn, conn->read_handler)) return;
       }
       /* Fire the writable event. */
       if (call_write) {
           if (!callHandler(conn, conn->write_handler)) return;
       }
       /* If we have to invert the call, fire the readable event now
        * after the writable one. */
       if (invert && call_read) {
           if (!callHandler(conn, conn->read_handler)) return;
       }
     }
   ```

2. connSocketEventHandler process connection, read, writing event unifiedly. It calls **readQueryFromClient** to handle client read event.
