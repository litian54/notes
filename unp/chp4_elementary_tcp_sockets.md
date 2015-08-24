## Chapter 4. Elementary TCP Sockets
### 'socket' Function
```c
#include <sys/socket.h>
int socket (int family, int type, int protocol);
/**
 * @family: protocal family
 *     AF_INET: IPv4 protocols (TCP|SCTP)
 *     AF_INET6: IPv6 procotols (TCP|SCTP)
 *     AF_LOCAL: Unix domain protocols
 *     AF_ROUTE: Routing Sockets (an interface to the kernel's routing table)
 *     AF_KEY: Key socket (an interface to the kernel's key table)
 *
 * @type: type of socket
 *     SOCK_STEAM: stream socket (TCP|SCTP)
 *     SOCK_DGRAM: datagram socket (UDP)
 *     SOCK_SEQPACKET: sequenced packet socket (SCTP)
 *     SOCK_RAM: raw socket (IPv4, IPv6)
 *
 * @protocal: protocal of sockets for AF_INET or AF_INET6
 *     IPPROTO_TCP: TCP transport protocal
 *     IPPROTO_UDP: UDP transport protocal
 *     IPPROTO_SCTP: SCTP transport protocal
 * Returns a smal non-negative integer value(socket descriptor or sockfd)
 */
```
AF_xxx: address family, used in socket address structure.
PF_xxx: protocol family, uesd to create the socket.
**Note**: single `PF_` --suport--> multiple `AF_`.(But nerver supported actually)

### 'connect' Function
```c
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
/**
 * @sockfd: a socket descriptor returned by function 'socket'
 * @servaddr: a pointer to server's socket address structure
 * @addrlen: the size of @servaddr
 * Returns: 0 if OK, -1 on error.
 *     different error returns possible:
 *         1. ETIMEDOUT: no response to its SYN segment.
 *         2. ECONNREFUSED: response RST(reset), means no process is waiting
 *                          for connection.
 *         3. EHOSTUNREACH or ENETUNREACH: the client's SYN elicits an ICMP
 *                                         "destination unreachable" from
 *                                          some intermediate router.
 */
```
> before calling connect: the kernel will choose both an ephemeral port and the source IP address if necessary.  

Three conditions that generate an RST:

* when a SYN arrives for a port that has no listening server
* when TCP wants to abort an existing connection
* when TCP receives a segment for a connection that does not exist

> time connect fails, we must close the socket descriptor and call socket again.

### 'bind' Function
```c
#include <sys/socket.h>
int bind (int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
/**
 * The bind function assigns a local protocol address to a socket.
 *
 * @myaddr: a pointer to a protocol-specific address
 * @addrlen: the size of myaddr
 * Returns: 0 if OK,-1 on error
 */
```
IPv4: wildcard address: INADDR_ANY(normally 0)
```c
struct sockaddr_in servaddr;
servaddr.sin_addr.s_addr = htonl (INADDR_ANY); /* wildcard */
```
IPv6: IPv6 address is stored in a structure. In C we cannot represent a constant structure on the right-hand side of an assignment.

```c
/**
 * <netinet/in.h> header contains the `extern` for in6addr_any
 */
struct sockaddr_in6 serv;
serv.sin6_addr = in6addr_any; /* wildcard */
```

> `bind` function not return ephemeral port number(if kernel to chooose port). We must call `getsockname` to return the protocal address.

Example for server end binds wildcard or non-wildcard address:

* non-wildcadr: multiple organizations, sanme subnet

    ```
    www.a.com -> 198.69.10.128 -+
    www.b.com -> 198.69.10.129 -|-> eth0(aliased -> single network interface)
    www.c.com -> 198.69.10.130 -+
    ......
    ```
* wildcard: When a connection arrives, the server calls getsockname to obtain the destination IP address from the client

### 'listen' Function
```c
#include <sys/socket.h>
int listen (int sockfd, int backlog);
/**
 * @backlog: the maximum number of connections the kernel should queue for
 *           this socket.
 * Returns: 0 if OK, -1 on error.
 */
```
`backlog`: the kernel maintains two queues:

1. An `incomplete connection queue`: contains an entry for each SYN that has arrived from a client for which the server is awaiting completion of the TCP three-way handshake.(These sockets are in the SYN_RCVD state)
2. A `completed connection queue`: contains an entry for each client that TCP three-way handshake has completed.(ESTABLESHED state)

> **Note:**

> * `backlog` >= sum of both queues.
> * When the process calls `accept`, the first entry on the completed queue is returned to the process, or if the queue is empty, the process is put to sleep until an entry is placed onto the completed queue.
> * Queues all full: when a client SYN arrives, TCP ignores the arriving SYN; it does not sent an RST. The client will retry(sent SYN) later.
> Date that arrives after 3-way handshake but before `accept`, should be queued by the server TCP, up to the size of the connected socket's recevice buffer.

`RTT`: an entry remains on the incomplete connection queue.

> `listen` `backlog` really means. It should specify the maximum number of completed connections for a given socket that the kernel will queue.

### 'accept' Function
```c
#include <sys/socket.h>
int accept (int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
/**
 * To return the next completed connection from the front of the completed
 * connection queue
 *
 * @sockfd: called 'listening socket', created and uesed by 'socket',
 *          'bind','listen'.
 * @cliaddr: used to return the protocol address of the connected peer
 *           process (the client). or Null.
 *
 * Returns: non-negative descriptor if OK, -1 on error
 *          the descriptor is called 'connected socket'.
 */
```
> **Note:** a server creates only one 'listening socket', the kernel creates one 'connected' socket for each client connection that is accepted

### 'fork' and 'exec' Functions
'fork':
```c
#include <unistd.h>
pid_t fork(void)
/**
 * Returns: 0 in child, process ID of child in parent, -1 on error
 */
> `fork` returns twice:
    * return child's pid (copy the current process as a child process)
    * return 0 (already in child process)
Therefore, if you call `fork` and the return value is 0, now you are in child.

```
`exec`:
```c
#include <unistd.h>
#include <unistd.h>
int execl (const char *pathname, const char *arg0, ... /* (char *) 0 */ );

int execv (const char *pathname, char *const argv[]);

int execle (const char *pathname, const char *arg0, .../* (char *) 0, char *const envp[] */ );

int execve (const char *pathname, char *const argv[], char *const envp[]);

int execlp (const char *filename, const char *arg0, ... /* (char *) 0 */ );

int execvp (const char *filename, char *const argv[]);

All six return: -1 on error, no return on success
```
relations of six `exec`
```
 execlp               execl           execle
   +                    +               +
   |                    |               |
   | create argv        | create argv   | create argv
   |                    |               |
   |                    |               |
   v    convert file    v     add       v    system
 execvp+------------->execv+--------->execve+------->
        to path               envp            call
```
### 'close' Function
```c
#include <unistd.h>
int close (int sockfd);
/** 
 * Just decrements the reference count for the descriptor
 * Returns: 0 if OK, -1 on error
 */
```
```
     parent
   +-----------+        +-----------+          +-----------+
   |           |        |           |          |           |
   | listenfd 1| fork   | listendf 2| close    | listenfd 1|
   |           +------->|           +--------->|           |
   | connfd   1|        | connfd   2| listenfd | connfd   1|
   |           |        |           | connfd   |           |
   +-----------+        +-----------+          +-----------+
```

### 'getsockname' and 'getpeername' Functions
```c
#include <sys/socket.h>
/**
 * Both return: 0 if OK, -1 on error
 * 'getsockname' retrun local protocol address associated with a socket
 * 'getpeername' retrun local foreign address associated with a socket
 *
 * if in server end: both function use 'connfd' (returned from 'accecpt')
 */
int getsockname(int sockfd, struct sockaddr *localaddr, socklen_t *addrlen)

int getpeername(int sockfd, struct sockaddr *peeraddr, socklen_t *addrlen)
```
