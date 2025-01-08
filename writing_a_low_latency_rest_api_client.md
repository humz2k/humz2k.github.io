# Writing a low latency REST API client (and why you would ever want to)

At some point recently I found myself looking for a low latency non-blocking REST API client library for C++. I found a bunch of stuff about writing low latency REST API servers, and a bunch of people asking why you would ever, ever, care about latency for a REST client. This is fair. Most of the time, latency is going to be dominated by the round trip time of the actual request, so what reasonable person would care about a few extra micro/milliseconds client side. Luckily, I am not a reasonable person.

### Exchange APIs for the average person

For actual low latency trading systems, there are special APIs (like FIX, or proprietary ones). These APIs are, however, not available for your average algotrading degenerate. Instead they have to make do with Websockets and REST. Most of the time, Websockets are just for streaming market data, which means you have to place/modify orders using a REST API. Now, most people using these APIs are *not* running super latency sensitive strategies. This is reasonable. There is no way you are going to be able to compete with someone using a binary API with Websockets and REST. Again, I am not a reasonable person.

### Kalshi

Kalshi is a event contract market that provides a REST API for interacting with the exchange, and a Websocket API for streaming market data (and a FIX API if you are [SIG](https://sig.com/)). This means that everyone (apart from SIG) is on a level playing field. There are also not that many people (yet) running automated trading systems on the exchange. So in this specific case, on this specific exchange, we might want a low latency REST API client.

### Curl

If I want to make REST API calls in C++, I'm probably going to use curl (or a curl wrapper like [restclient-cpp](https://github.com/mrtazz/restclient-cpp)). Curl in its normal mode is **blocking**. This doesn't work for a low latency trading system. We don't want to pause the entire system while we wait to place or cancel an order. Curl has a multi mode where you can have non-blocking requests, but it has a pretty gross interface, and we don't want to be dealing with threads and locks anyway, because this adds overhead. So what is the solution?

### HTTP and Sockets and SSL/TLS

HTTP is a pretty simple protocol (at least HTTP 1.1 is). All we are doing is sending and receiving text from a server. So we could just use raw non-blocking sockets and do all the HTTP parsing and constructing of requests ourselves. We also need to deal with SSL/TLS stuff, but this is pretty simple with OpenSSL.

A sketch of our system might look like this:
* Open a non-blocking socket connection to the server, and keep it alive.
* When we want to make a request, we construct the text of the request, and send it (non-blocking) using the socket.
* We poll the socket to listen for responses.

## A Naive Implementation

Lets build a extremely simple version of this idea.

*Disclaimer: the code below might not compile - I'm grabbing bits and pieces from my final implementation, and am not checking for typos.*

As always with sockets, the boiler plate is super annoying. We first need to figure out what includes we need.

```c++
#include <iostream> // debugging
#include <string> // gonna be using these a bunch

// socket and SSL garbage
#include <arpa/inet.h>
#include <fcntl.h>
#include <netdb.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <openssl/err.h>
#include <openssl/ssl.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
```

The interface of the client is going to look something like this

```c++
class SocketClient {
  public:
    SocketClient(const std::string host, const long port = 443);
    int send_request(const std::string& req);
    std::string read_buffer(const size_t read_size = 100);
};
```

and we are going to need to keep track of a few things.

```c++
// the url of the host we are making requests to
std::string m_host;

// socket
int m_sockfd = -1;

// ssl socket, the thing we actually use
int m_sslsock = -1;

// ssl stuff
SSL_CTX* m_ctx = nullptr;
SSL* m_ssl = nullptr;
```

The constructor will connect to the host, and is exclusively sockets/SSL boilerplate. This isn't a socket tutorial, so I'll just put the code here.

```c++
// dumb way to print ssl errors
std::string get_ssl_error() {
    std::string out = "";
    int err;
    while ((err = ERR_get_error())) {
        char* str = ERR_error_string(err, 0);
        if (str)
            out += std::string(str);
    }
    return out;
}

SocketClient(const std::string host, const long port = 443) : m_host(host) {
    // lots of boring socket config stuff, so much boilerplate
    struct addrinfo hints = {}, *addrs;

    hints.ai_family = AF_INET;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP;

    if (int rc = getaddrinfo(host.c_str(), std::to_string(port).c_str(),
                                &hints, &addrs);
        rc != 0) {
        throw SocketClientException(std::string(gai_strerror(rc)));
    }

    for (addrinfo* addr = addrs; addr != NULL; addr = addr->ai_next) {
        m_sockfd =
            socket(addr->ai_family, addr->ai_socktype, addr->ai_protocol);

        if (m_sockfd == -1)
            break;

        // set nonblocking socket flag
        int flag = 1;
        if (setsockopt(m_sockfd, IPPROTO_TCP, TCP_NODELAY, (char*)&flag,
                        sizeof(int)) < 0) {
            std::cerr << "Error setting TCP_NODELAY" << std::endl;
        } else {
            if constexpr (verbose) {
                std::cout << "Set TCP_NODELAY" << std::endl;
            }
        }

        // try and connect
        if (connect(m_sockfd, addr->ai_addr, addr->ai_addrlen) == 0)
            break;

        close(m_sockfd);
        m_sockfd = -1;
    }

    if (m_sockfd == -1)
        throw SocketClientException("Failed to connect to server.");

    // ssl boilerplate
    const SSL_METHOD* meth = TLS_client_method();
    m_ctx = SSL_CTX_new(meth);
    m_ssl = SSL_new(m_ctx);

    if (!m_ssl)
        throw SocketClientException("Failed to create SSL.");

    SSL_set_tlsext_host_name(m_ssl, host.c_str());

    m_sslsock = SSL_get_fd(m_ssl);
    SSL_set_fd(m_ssl, m_sockfd);

    if (SSL_connect(m_ssl) <= 0) {
        throw SocketClientException(get_ssl_error());
    }

    if constexpr (verbose) {
        std::cout << "SSL connection using " << SSL_get_cipher(m_ssl)
                    << std::endl;
    }

    freeaddrinfo(addrs);

    fcntl(m_sockfd, F_SETFL, O_NONBLOCK);
}
```

The destructor then needs to clean all of this up. More boringness.

```c++
~SocketClient() {
    if (!(m_sockfd < 0))
        close(m_sockfd);
    if (m_ctx)
        SSL_CTX_free(m_ctx);
    if (m_ssl) {
        SSL_shutdown(m_ssl);
        SSL_free(m_ssl);
    }
}
```

Then because we are careful programmers, we delete copy constructors and write move constructors. This step is optional.

```c++
SocketClient(const SocketClient&) = delete;
SocketClient& operator=(const SocketClient&) = delete;

SocketClient(SocketClient&& other)
    : m_host(std::move(other.m_host)), m_sockfd(other.m_sockfd),
        m_sslsock(other.m_sslsock), m_ctx(other.m_ctx), m_ssl(other.m_ssl) {
    other.m_sockfd = -1;
    other.m_sslsock = -1;
    other.m_ctx = nullptr;
    other.m_ssl = nullptr;
}

SocketClient& operator=(SocketClient&& other) {
    if (!(m_sockfd < 0))
        close(m_sockfd);
    if (m_ctx)
        SSL_CTX_free(m_ctx);
    if (m_ssl) {
        SSL_shutdown(m_ssl);
        SSL_free(m_ssl);
    }

    m_host = std::move(other.m_host);
    m_sockfd = other.m_sockfd;
    m_sslsock = other.m_sslsock;
    m_ctx = other.m_ctx;
    m_ssl = other.m_ssl;

    other.m_sockfd = -1;
    other.m_sslsock = -1;
    other.m_ctx = nullptr;
    other.m_ssl = nullptr;
}
```

Everything else is pretty simple:
```c++
// sends a request - forces the socket to fully send everything
int send_request(const std::string& req) {
    const char* buf = req.c_str();
    int to_send = req.size();
    int sent = 0;
    while (to_send > 0) {
        const int len = SSL_write(m_ssl, buf + sent, to_send);
        if (len < 0) {
            int err = SSL_get_error(m_ssl, len);
            switch (err) {
            case SSL_ERROR_WANT_WRITE:
                throw SocketClientException("SSL_ERROR_WANT_WRITE");
            case SSL_ERROR_WANT_READ:
                throw SocketClientException("SSL_ERROR_WANT_READ");
            case SSL_ERROR_ZERO_RETURN:
                throw SocketClientException("SSL_ERROR_ZERO_RETURN");
            case SSL_ERROR_SYSCALL:
                throw SocketClientException("SSL_ERROR_SYSCALL");
            case SSL_ERROR_SSL:
                throw SocketClientException("SSL_ERROR_SSL");
            default:
                throw SocketClientException("UNKNOWN SSL ERROR");
            }
        }
        to_send -= len;
        sent += len;
    }
    return sent;
}

// read whatever is in the recv buffer right now
std::string read_buffer(const size_t read_size = 100) {
    size_t read = 0;
    std::string out;
    while (true) {
        const size_t original_size = out.size();
        out.resize(original_size + read_size);
        char* buf = &(out.data()[original_size]);
        int rc = SSL_read_ex(m_ssl, buf, read_size, &read);
        out.resize(original_size + read);
        if ((read < read_size) || (rc == 0)) {
            break;
        }
    }
    return out;
}
```

This is already pretty good (ignoring the fact that we aren't parsing the response), but there is one big thing that we will probably need if we want to improve latency.

## String concatenation

Heap allocations in the hot path will kill you in terms of latency. So we want to minimize, or hopefully eliminate, heap allocations. For REST request/response construction and parsing, we are going to need to concatenate strings a bunch. The low hanging fruit here is to just use a string type with a pool allocator, and just do `str + str` for concatenation, but we can do better.

What we really need is a bunch of preallocated memory, and then when we construct a request or parse a response, we just copy stuff into this preallocated memory. For example, I could have a buffer of size 10. Then to concatenate `"abc"` and `"def"`, I copy `"abc"` into `buffer[0:3]`, and `"def"` into `buffer[3:6]`, and return a `string_view` from `buffer[0:6]`. Hard coding this stuff wherever its necessary wouldn't be too hard, but we might as well make something reusable.

This is what I ended up coming up with:

```c++
#include <iostream>
#include <memory>
#include <sstream>
#include <string>
#include <vector>

namespace smallstring {

template <typename RawBufferType = std::vector<char>> class Buffer {
  private:
    std::size_t m_length = 0;
    RawBufferType m_buffer;

  public:
    Buffer(std::size_t capacity = 256) : m_buffer(capacity) {}
    Buffer(Buffer&& other)
        : m_length(other.m_length), m_buffer(std::move(other.m_buffer)) {
        other.m_length = 0;
    }
    Buffer& operator=(Buffer&& other) {
        m_length = other.m_length;
        other.m_length = 0;
        m_buffer = std::move(other.m_buffer);
        return *this;
    }

    // deleted copy constructors
    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;

    std::size_t capacity() const { return m_buffer.size(); }

    std::size_t length() const { return m_length; }

    std::size_t remaining() const { return capacity() - length(); }

    void drop_memory() {
        m_buffer.clear();
        m_buffer.shrink_to_fit();
        m_length = 0;
    }

    void ensure_fit(const std::size_t to_add) {
        const std::size_t new_capacity = m_length + to_add;
        if (new_capacity > capacity()) {
            m_buffer.resize(new_capacity);
        }
    }

    void ensure_pad(const std::size_t pad) { ensure_fit(pad); }

    char* head() { return m_buffer.data(); }
    const char* head() const { return m_buffer.data(); }

    char* tail() { return m_buffer.data() + m_length; }
    const char* tail() const { return m_buffer.data() + m_length; }

    char* begin() { return head(); }
    const char* begin() const { return head(); }

    char* end() { return tail(); }
    const char* end() const { return tail(); }

    void clear() { m_length = 0; }

    void push(const char* ptr, std::size_t sz) {
        ensure_fit(sz);
        std::copy(ptr, ptr + sz, tail());
        m_length += sz;
    }

    template <std::size_t N> void push(const char (&ptr)[N]) {
        push(ptr, N - 1);
    }

    void push(const std::string_view& view) {
        push(view.data(), view.length());
    }

    void push(const std::string& str) { push(str.data(), str.length()); }

    void pop(const std::size_t n) {
        if (n >= m_length) {
            m_length = 0;
            return;
        }
        std::copy(head() + n, tail(), head());
        m_length = m_length - n;
    }

    std::string_view view() {
        return std::string_view(m_buffer.data(), m_length);
    }

    const std::string_view view() const {
        return std::string_view(m_buffer.data(), m_length);
    }

    std::size_t find(const char* str) const { return view().find(str); }

    std::size_t find(const std::string& str) const { return view().find(str); }
};

} // namespace smallstring
```