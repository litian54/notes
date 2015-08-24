## Chapter 6. I/O Multiplexing: The 'select' and 'poll' Functions
I/O multiplexing is typically used in networking applications in the following scenarios:

* a client is handling multiple descriptors (input and network socket)
* a client to handle multiple sockets
* a TCP server handles both a listening socket and its connected sockets
* a server handles both TCP and UDP
* a server handles multiple services and perhaps multiple protocols (inetd daemon)

### I/O Models
five I/O models:

* blocking I/O
* nonblocking I/O
* I/O multiplexing (select and poll)
* signal driven I/O (SIGIO)
* asynchronous I/O (the POSIX aio_functions)

**Input** operations:

* Waiting for the data to be ready (data from network -> kernel buffer)
* Copying the data from the kernel to the process (kernel buffer -> application buffer)

#### Blocking I/O Model
UDP for Blocking I/O
```
               applications                  kernel
                          system call
          +--  recvfrom +-------------> no datagram ready  --+
          |                                    +             |
          |                                    |             |
          |                                    |             >wait for data
          |                                    |             |
          |                                    |             |
          |                                    v             |
process   |                               datagram ready   --+
blocks in <
call to   |                               copy datagram    --+
recvfrom  |                                    +             |
          |                                    |             |
          |                                    |             > copy data from
          |                                    |             | kernel to user
          |                                    |             |
          |               retrun OK            v             |
          +--  process  <--------------+  copy complete    --+
               datagram
```
#### Nonblocking I/O Model
The I/O operation has to put the process to sleep, do not put the process to sleep, but return an error instead.  
**Polling** (a loop calling recvfrom on a nonblocking descriptor)
```
               applications                  kernel
                          system call
          +--  recvfrom +-------------> no datagram ready  --+
          |               EWOULDBLOCK                        |
          |             <-------------+                      |
          |               system call                        >wait for data
          |    recvfrom +-------------> no datagram ready    |
          |               EWOULDBLOCK                        |
          |             <-------------+                      |
          |               system call                        |
process   |    recvfrom +------------->   datagram ready   --+
repeatedly<
calls to  |                               copy datagram    --+
recvfrom, |                                    +             |
waiting   |                                    |             |
for an OK |                                    |             > copy data from
retrun    |                                    |             | kernel to user
(polling) |                                    |             |
          |               retrun OK            v             |
          +--  process  <--------------+  copy complete    --+
               datagram
```
This is often a waste of CPU time. This model is normally on systems dedicated to one function.

#### I/O Multiplexing Model
```
                  applications                  kernel
                             system call
             +--  select   +-------------> no datagram ready  --+
             |                                    +             |
process      |                                    |             |
blocks in    |                                    |             |
call to      |                                    |             |
select,      <                                    |             > wait
waiting for  |                                    |             | for data
one of       |                                    |             |
possibly many|                                    |             |
sockets to   |                                    |             |
become       |              retrun readable       v             |
readable     +--           <---------------+ datagram ready   --+
                            system call
             +--  recvfrom +---------------> copy datagram    --+
process      |                                    +             |
blocks while |                                    |             |
data copied  <                                    |             > copy data
into         |                                    |             | from kernel
application  |                                    |             | to user
buffer       |               retrun OK            v             |
             +--  process  <--------------+  copy complete    --+
                  datagram
```
`select` can wait for more than one descriptor to be ready.

#### Signal-Driven IO Model
telling the kernel to notify us with the `SIGIO` signal when the descriptor is ready.
```
                     applications                     kernel
                                sigaction system call
               establish SIGIO  +-------------------->
            +-- signal handler  <--------------------+          --+
            |                                                     |
            |                                                     |
process     |                                                     |
continues   <                                                     > wait
executing   |                                                     | for data
            |                                                     |
            +--                                                   |
                                  deliver SIGIO                   |
            +-- signal handler <---------------- datagram ready --+
            |                       system call
            |    recvfrom       +---------------> copy datagram --+
process     |                                       +             |
blocks      |                                       |             |
while data <                                        |             > copy data
copied into |                                       |             | from
application |                                       |             | kernel to
buffer      |                   retrun OK           v             | user
            +--  process       <--------------+  copy complete  --+
                datagram
```
when the datagram is ready to be read, the SIGIO signal is generated for our process:

* read data from the signal handler by calling recvfrom -> notify the main loop that the data is ready
* can notify the main loop and let it read the datagram

#### Asynchronous I/O Model
Difference from signal-driven I/O:

    * signal-driven I/O: the kernel tells us when an I/O operation can be initiated.
    * asynchronous I/O: the kernel tells us when an I/O operation is complete.

```
               applications                  kernel
                          system call
          +--  aio_read +-------------> no datagram ready  --+
          |             <-------------+        +             |
          |                 return             |             |
          |                                    |             >wait for data
          |                                    |             |
          |                                    |             |
          |                                    v             |
process   |                               datagram ready   --+
continues <
executing |                               copy datagram    --+
          |                                    +             |
          |                                    |             |
          |                                    |             > copy data from
          |                                    |             | kernel to user
          |                                    |             |
          | signal handle  deliver signal      v             |
          +--  process  <---------------+ copy complete    --+
               datagram  specified in aio_read
```

#### Synchronous I/O versus Asynchronous I/O
POSIX defines:

    * A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes. (blocking, noblocking, I/O multiplexing, signal-driven I/O)
    * An asynchronous I/O operation does not cause the requesting process to be blocked. (asynchronous I/O)

### 'select' Function
```c
#include <sys/select.h>
#include <sys/time.h>
/**
 * @maxfdp1: argument specifies the number of descriptors to be tested.
 *           Its value is the maximum descriptor to be tested plus one
 *           (hence our name of maxfdp1).
 *           for example: descriptors 1, 4, 5 -> maxfdp1 is 6
 * @exceptset:
 *     1. The arrival of out-of-band data for a socket.
 *     2. pseudo-terminal
 *
 * Returns: positive count of ready descriptors, 0 on timeout, –1 on error
 */
int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset,
const struct timeval *timeout);

/**
 * Three possibilities for timeval:
 *     1. null pointer: wait forever untill one fd is ready for I/O
 *     2. wait up to a fixed amount of time
 *     3. tv_sec and tv_usec are 0: do not wait at all. called `polling`.
 */
struct timeval {
    long tv_sec; /* seconds */
    long tv_usec; /* microseconds */
};

void FD_ZERO(fd_set *fdset); /* clear all bits in fdset */
void FD_SET(int fd, fd_set *fdset); /* turn on the bit for fd in fdset */
void FD_CLR(int fd, fd_set *fdset); /* turn off the bit for fd in fdset */
int FD_ISSET(int fd, fd_set *fdset); /* is the bit for fd on in fdset ? */

/**
 * fd_set is like (an array of integers):
 * An array   +--------------------------------......
 *            |   |   |   |   |   |   |   |   |......
 *            +-|------------------------------......
 *              | (32 bit integer)
 *              v
 *  0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 1 (value 5)
 *  (represent descriptor 0 - 31)
 *
 * corresponding: descriptor 0 and descriptor 2
 */
fd_set rset;

FD_ZERO(&rset); /* initialize the set: all bits off */
FD_SET(1, &rset); /* turn on bit for fd 1 */
FD_SET(4, &rset); /* turn on bit for fd 4 */
FD_SET(5, &rset); /* turn on bit for fd 5 */
```
> if readset, writeset and exceptset are all null, the `select`(microsecond) is a higher precision `sleep`(second).

* The constant `FD_SETSIZE`, defined by including `<sys/select.h>`, is the number of descriptors in the fd_set datatype. (often 1024)
* We use the FD_ISSET macro on return to test a specific descriptor in an fd_set structure. Any descriptor that is not ready on return will have its corresponding bit cleared in the descriptor set.

common programming errors when using `select`:

* forget to add one to the largest descriptor number and fd_set are value-result.
* set bit to 0 which should be 1.

A socket is ready for reading:

* The number of bytes of data in the socket receive buffer is greater than or equal to the current size of the low-water mark for the socket receive buffer. `SO_RCVLOWAT` socket option for seting low-water.
* The read half of the connection is closed (i.e., a TCP connection that has received a FIN). A read operation on the socket will not block and will return 0 (i.e., EOF).
* listening socket and completed connections is nonzero. (new connection is ready)
* A socket error is pending. A read operation on the socket will not block and will return an error (–1) with errno set to the specific error condition. These pending errors can also be fetched and cleared by calling `getsockopt` and specifying the `SO_ERROR` socket option.

A socket is ready for writing:

* The number of bytes of available space in the socket send buffer is greater than or equal to the current size of the low-water mark for the socket send buffer and either: (i) the socket is connected, or (ii) the socket does not require a connection (e.g., UDP). `SO_SNDLOWAT` socket option for setting low-water. (normally 2048)
* The write half of the connection is closed. A write operation on the socket will generate `SIGPIPE`.
* A socket using a non-blocking connect has completed the connection, or the connect has failed.
* A socket error is pending. A write operation on the socket will not block and will return an error (–1) with errno set to the specific error condition. These pending errors can also be fetched and cleared by calling getsockopt with the SO_ERROR socket option.

A socket has an exception:

* there is out-of-band data for the socket
* the socket is still at the out-of-band mark

Example of `select`
```c
#include        "unp.h"

void
str_cli(FILE *fp, int sockfd)
{
        int                     maxfdp1;
        fd_set          rset;
        char            sendline[MAXLINE], recvline[MAXLINE];

        FD_ZERO(&rset);
        for ( ; ; ) {
                FD_SET(fileno(fp), &rset);
                FD_SET(sockfd, &rset);
                maxfdp1 = max(fileno(fp), sockfd) + 1;
                Select(maxfdp1, &rset, NULL, NULL, NULL);

                if (FD_ISSET(sockfd, &rset)) {  /* socket is readable */
                        if (Readline(sockfd, recvline, MAXLINE) == 0)
                                err_quit("str_cli: server terminated prematurely");
                        Fputs(recvline, stdout);
                }

                if (FD_ISSET(fileno(fp), &rset)) {  /* input is readable */
                        if (Fgets(sendline, MAXLINE, fp) == NULL)
                                return;         /* all done */
                        Writen(sockfd, sendline, strlen(sendline));
                }
        }
}
```

