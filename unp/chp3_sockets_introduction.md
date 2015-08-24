## Chapter 4. Sockets Introduction
### socket functions
Pass(by reference) a socket address structure:

* process --> kernel: `bind`, `connect`, `sendto`, `sendmsg`(all go through the sockargs)
* kernel --> process: `accept`, `recvfrom`, `recvmsg`, `getpeername`, `getsockname`

IPv4 socket address structure:
<netinet/in.h>
```c
struct in_addr {
    in_addr_t s_addr;  /* 32-bit IPv4 address */
                       /* network byte ordered */
};

struct sockaddr_in {
    uint8_t sin_len;  /* length of structure (16) */
                      /* "sin_" means socket intnet */
    sa_family_t sin_family;  /* AF_INET */
    in_port_t sin_port;  /* 16-bit TCP or UDP port number */
                         /* network byte ordered */
    struct in_addr sin_addr;  /* 32-bit IPv4 address */
                              /* network byte ordered */
    char sin_zero[8];  /* unused */
};
```

IPv6 socket address structure:
<netinet/in.h>
```c
struct in6_addr {
    uint8_t s6_addr[16];  /* 128-bit IPv6 address */
                          /* network byte ordered */
};

#define SIN6_LEN /* required for compile-time tests */

struct sockaddr_in6 {
    uint8_t sin6_len;  /* length of this struct (28) */
    sa_family_t sin6_family;  /* AF_INET6 */
    in_port_t sin6_port;  /* transport layer port# */
                          /* network byte ordered */
    uint32_t sin6_flowinfo;  /* flow information, undefined */
    struct in6_addr sin6_addr;  /* IPv6 address */
                                /* network byte ordered */
    uint32_t sin6_scope_id;  /* set of interfaces for a scope */
};
```

The POSIX specification requires: `sin_family`, `sin_addr` and `sin_port`.

### Generic Socket Address Structure
**Note**: in ANSI C, `void *` is the generic pointer type.

Old:
<sys/socket.h>
```c
struct sockaddr {
    unit8_t sa_len;
    sa_family_t sa_family;  /* address family: AF_xxx value */
    char sa_data[14];  /* protocol-specific address */
}
```
New:
<netinet/in.h>
```c
struct sockaddr_storage {
    uint8_t ss_len;  /* length of this struct (implementation
                      * dependent)
                      */
    sa_family_t ss_family; /* address family: AF_xxx value */
    /* implementation-dependent elements to provide:
     * a) alignment sufficient to fulfill the alignment requirements of
     * all socket address types that the system supports.
     * b) enough storage to hold any type of socket address that the
     * system supports.
     */
};
```
The sockaddr_storage type provides a generic socket address structure that is different from struct sockaddr in two ways:
a. If any socket address structures that the system supports have alignment requirements, the sockaddr_storage provides the strictest alignment requirement.
b. The sockaddr_storage is large enough to contain any socket address
structure that the system supports.

### value-result argument
For functions:

* process --> kernel: `bind`, `connect`, `sendto`, `sendmsg`(all go through the sockargs)
Two arguments are:
    * a pointer to socket address structure
    * an integer with the value of the size of the structure
    ```c
    struct sockaddr_in serv;

    /* fill in serv{} */
    connect (sockfd, (SA *) &serv, sizeof(serv));
    ```

* kernel --> process: `accept`, `recvfrom`, `recvmsg`, `getpeername`, `getsockname`
Two arguments are:
    * a pointer to socket address structure
    * a pointer to an integer containing the size of the structure(**value-result**)
    ```c
    struct sockaddr_un cli; /* Unix domain */
    socklen_t len;
    
    len = sizeof(cli); /* len is a value */
    getpeername(unixfd, (SA *) &cli, &len);
    /* len may have changed */
    ```

The reason to use value-result pointer is:

* value: when function called, tells the kernel socket address structure size.
* result: function to return, tells the process the size the kernel actuall stored in structure.

**Note:** in `recvmsg` and `sendmsg`, length field is not an argument, but is a structure member.

### Byte Ordering Functions
* `little-endian` or `host byte order`: stored at the starting address of the value from low to high memory address.(从内存地址低位开始存值)
* `big-endian` or `network byte order`: stored at the starting address of the value from high to low memory address.(从内存地址高位开始存值)
```c
#include "unp.h"
int
main(int argc, char **argv)
{
    union {
        short s;
        char c[sizeof(short)];
    } un;
    un.s = 0x0102;
    printf("%s: ", CPU_VENDOR_OS);
    if (sizeof(short) == 2) {
        /* 大端: 因为un.c[1]是高地址位, 存的是un.s的起始值 */
        if (un.c[0] == 1 && un.c[1] == 2)
            printf("big-endian\n");
        /* 小端: 因为un.c[0]是低地址位, 存的是un.s的起始值 */
        else if (un.c[0] == 2 && un.c[1] == 1)
            printf("little-endian\n");
        else
            printf("unknown\n");
    } else
        printf("sizeof(short) = %d\n", sizeof(short));
    exit(0);
}
```

**Note:** `octet` == `byte` == `8-bit quantity`

### Bytes Manipulation Functions
begin with `b`(for byte)
```C
#include <strings.h>

/**
 * use to initialize a socket address structure to 0.
 */
void bzero(void *dest, size_t nbytes);

/**
 * move @nbytes from @src to @dest
 * can handle fields overlapping
 */
void bcopy(const void *src, void *dest, size_t nbytes);

/**
 * compare @nbytes strings of @ptr1 and @ptr2
 * Returns
 *     @ptr1 > @ptr2: > 0
 *     @ptr1 = @ptr2: 0
 *     @ptr1 < @ptr2: < 0
 */
int bcmp(const void *ptr1, const void *ptr2, size_t nbytes);
```

begin with `mem`(for memory)
```c
#include <string.h>

/**
 * set @len bytes of value @c to @dest
 */
void *memset(void *dest, int c, size_t len);

/**
 * similar to "bcopy", but arguments is swapped, like,
 * @dest = @src
 * is undefined for fields of @dest and @src overlapping
 */
void *memcpy(void *dest, const void *src, size_t nbytes);

/**
 * compare @nbytes strings of @ptr1 and @ptr2
 * Returns
 *    @ptr1 > @ptr2: > 0
 *    @ptr1 = @ptr2: 0
 *    @ptr1 < @ptr2: < 0
 *  The comparison is done assuming the two unequal bytes are unsigned chars
 */
int memcmp(const void *ptr1, const void *ptr2, size_t nbytes);
```
### 'inet_aton', 'inet_addr', and 'inet_ntoa' Functions
convert IPv4 Internet addresses (e.g., "206.168.112.96")<--> ASCII strings
```c
#include <arpa/inet.h>

/*  covert @strptr(e.g., "192,168,1,1") to
 *  @addrptr(32-bit binary network byte ordered value)
 *  return:
 *      valid: 1
 *      error: 0
 */
int inet_aton(const char *strptr, struct in_addr *addrptr);

/* Deprecated !!
 * Returns
 *     32-bit binary network byte ordered IPv4 address;
 *     `INADDR_NONE` if error
 *      some return `1` instead of `INADDR_NONE`
 */
in_addr_t inet_addr(const char *strptr);

/* convert a 32-bit binary network byte ordered IPv4 address
 * into its corresponding dotted-decimal string.
 * the return value resides in static memory
 * Note: the function takes a structure as its arg, not a pointer.
 */
char *inet_ntoa(struct in_addr inaddr);
Returns: pointer to dotted-decimal string
```

### 'inet_pton' and 'inet_ntop' Functions
Both worked for IPv6 and IPv4.  

* `p`: presentation, often an ASCII string.
* `n`: numeric, binary value that stored in socket address structure.

```c
#include <arpa/inet.h>

/**
 * @family: `AF_INET` or `AF_INT6`
 * @strptr: pointer to presentation format string
 * @addrptr: 
 * Returns 1 if OK, 0 if input not a valid presentation format, -1 on error
 */
int inet_pton(int family, const char *strptr, void *addrptr);

/**
 * Return pointer to result if OK, NULL on error
 *
 * @len: is the size of the @strptr
 * to prevent overflowing the caller's buffer
 * 
 * If @len is too small to hold the resulting presentation format,
 * including the terminating null, a null pointer is returned
 * and `errno` is set to `ENOSPC`
 */
const char *inet_ntop(int family, const void *addrptr, char *strptr, size_t len);

/**
 * If @family is not supported, 
 * both functions return an error with errno set to `EAFNOSUPPORT`
 */
```

### 'sock_ntop' and related Functions
```c
#include "unp.h"

/**
 * Returns non-null pointer if OK, NULL on error
 * @sockaddr points to a socket address structure whose length is @addrlen
 * The function uses its own static buffer to hold the result and a pointer
 * to this buffer is the return value.
 *
 * The presentation format:
 * xxx.xxx.xxx.xxx:port(decimal)'\0'
 * or
 * [hex string form of IPv6]:port(decimal)'\0'
 *
 * So the buffer size is at least(bytes):
 * IPv4: INET_ADDRSTRLEN('xxx.xxx.xxx.xxx'+'\0'=16) + 6(port+':') = 22
 * IPv6: INET6_ADDRSTRLEN(hex string+'\0'=46) + 8(port+':'+'[]')= 54
 */
char *sock_ntop(const struct sockaddr *sockaddr, socklen_t addrlen);
```

### 'readn', 'writen', and 'readline' Functions
```c
/* include readline */
#include        "unp.h"

static int      read_cnt;
static char     *read_ptr;
static char     read_buf[MAXLINE];

static ssize_t
my_read(int fd, char *ptr)
{
    /* 通过read_cnt判断是读入还是返回一个char */
    if (read_cnt <= 0) {
        again:
        if ( (read_cnt = read(fd, read_buf, sizeof(read_buf))) < 0) {
            if (errno == EINTR)
                goto again;
                return(-1);
            } else if (read_cnt == 0)
                return(0);
            read_ptr = read_buf;
    }

    read_cnt--;
    /* ptr每次只指向一个char */
    *ptr = *read_ptr++; //先赋值, 后自加
    return(1);
}

ssize_t
readline(int fd, void *vptr, size_t maxlen)
{
    ssize_t n, rc;
    char    c, *ptr;

    ptr = vptr;
    /**
     * my_read一次性读入MAXLINE的字符串, 然后一个一个的返回char
     * 主要通过static的read_cnt来控制是读入还是返回一个char
     */
    for (n = 1; n < maxlen; n++) {
        if ( (rc = my_read(fd, &c)) == 1) {
            *ptr++ = c;
            if (c == '\n')
                break;  /* newline is stored, like fgets() */
                } else if (rc == 0) {
                    *ptr = 0;
                    return(n - 1);  /* EOF, n - 1 bytes were read */
                } else
                    return(-1);             /* error, errno set by read() */
    }

    *ptr = 0;       /* null terminate like fgets() */
    return(n);
}
```
