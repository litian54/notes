## Chapter 5. TCP Client/Server Example
### POSIX Signal Handling
Signals can be sent:

    * By one process to another process(or to itself)
    * By the kernel to a process

Set signal's "disposition"(or "action") by calling the `sigaction` function:

    * "signal handler": a function to be called whenever a specific singal occurs. This action is called catching a signal.
    * Ingore a signal by setting its disposition to `SIG_IGN`. `SIGKILL` and `SIGSTOP` cannot by ingored.
    * set the default disposition for a signal by setting its disposition to `SIG_DFL`

### POSIX Signal Handling
```c
void (*signal (int signo, void (*func)(int))) (int);
```
To understand the structure above,
```c
typedef void Sigfunc(int);
```

* `Sigfunc` type is a function has an integer argument and returns nothing(void).
* `void (*fun)(int)` is a pointer to a function, that function matches the structre of `Sigfunc`.

So the function prototype becomes,
```c
Sigfunc *signal (int signo, Sigfunc *func);
```
signal function code
```c
#include        "unp.h"

Sigfunc *
signal(int signo, Sigfunc *func)
{
        struct sigaction        act, oact;

        act.sa_handler = func;
        /**
         * 'sa_mask' is used to specify a set of signals that will be blocked
         * when our signal handler is called.
         *
         * 'sigemptyset' sets the 'sa_mask' member to the empty set, means
         * that no additional signals will be blocked while our signal
         * handler is running.
         *
         * POSIX guarantees that the signal being caught is always blocked
         * while its handler is executing.
         */
        sigemptyset(&act.sa_mask);
        act.sa_flags = 0;
        if (signo == SIGALRM) {
#ifdef  SA_INTERRUPT
                act.sa_flags |= SA_INTERRUPT;   /* SunOS 4.x */
#endif
        } else {
#ifdef  SA_RESTART
                act.sa_flags |= SA_RESTART;             /* SVR4, 44BSD */
#endif
        }
        /**
         * set new action('act'), return old action to 'oact'.
         */
        if (sigaction(signo, &act, &oact) < 0)
                return(SIG_ERR);
        return(oact.sa_handler);
}
```

### Handling 'SIGCHLD' Signals
"slow system call": Most networking functions.  
The basic rule is, for example:
```
  Parent P:  accept(blocked)+---->returns an error EINTR
                   ^
                   |   interrupted
                   +
             Signal(SIGCHLD, sig_chld);
                   ^
                   |   catched
                   +
  Child P:  SIGCHLD
```
To hanld an interrupted,
```c
for ( ; ; ) {
    clilen = sizeof (cliaddr);
    if ( (connfd = accept (listenfd, (SA *) &cliaddr, &clilen)) < 0) {
        if (errno == EINTR)
            continue; /* back to for () */
        else
            err_sys ("accept error");
}
```
### 'wait' and 'waitpid' Functions
```c
#include <sys/wait.h>
/**
 * wait blocks until the first of the existing children terminates.
 * @statloc: a pointer to an integer of terminated status.
 * Returns: process ID of the terminated child if OK, 0 or –1 on error
 */
pid_t wait (int *statloc);

/**
 * more control for us
 * @pid: which proccess to wait. '-1' to wait the first child.
 * @statloc: a pointer to an integer of terminated status.
 * @options: usually 'WNOHANG', tells kernel not to block if there are no
 *           terminated children.
 * Returns: process ID of the terminated child if OK, 0 or –1 on error
 */
pid_t waitpid (pid_t pid, int *statloc, int options);
```

### Connection Abort before 'accept' Retruns
On an ESTABLISHED connection, the client sent RST before the server calls `accept`.(Section 16.6)

### Termination of Server Process
On an ESTABLISHED connection:
```
                                       recv:ACK
     server: kill child ---> FIN_WAIT_1 ----->FIN_WAIT2 
                         |                ^
(call 'close') send: FIN |                | sent ACK
                         v                |
     clinet:       ESTABLISHED ----------------------> CLOSE_WAIT (blocked)
                                  recv:FIN
```
should use `select` or `poll`.

### 'SIGPIPE' Signal
The rule that applies is: When a process writes to a socket that has received an RST, the `SIGPIPE` signal is sent to the process. The default action of this signal is to terminate the process, so the process must catch the signal to avoid being involuntarily terminated.

> It is okay to write to a socket that has received a FIN, but it is an error to write to a socket that has received an RST.

### Crashing of Server Host
For example, disconnect the server host from the network.

### Crashing and Rebooting of Server Host
For example,

1. establish the connection, disconnect the server from the network, shut down the server host and then reboot it, and then reconnect the server host to the network.
2. After the server rebooted, its TCP loses all information about the previous connection. The server will responese an RST.
3. The client received the RST, return `ECONNRESET`.

### Shutdown of Server Host
Unix system shut down:

1. the init process normally sends the SIGTERM signal to all processes (we can catch this signal)
2. then sends the SIGKILL signal(which cannot catch)

If we do not catch SIGTERM and terminate, our server will be terminated by the SIGKILL signal.
