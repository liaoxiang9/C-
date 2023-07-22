# C-
```C++
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <string.h>
#include <iostream>
#include <vector>
#include <algorithm>

class EventHandler {
public:
    virtual void handle_read() = 0;
    virtual int get_handle() = 0;
};

class EchoHandler : public EventHandler {
    int clnt_sock;

public:
    EchoHandler(int sock) : clnt_sock(sock) {}

    int get_handle() {
        return clnt_sock;
    }

    void handle_read() {
        char buffer[1024];
        int str_len = read(clnt_sock, buffer, sizeof(buffer) - 1);
        if(str_len == 0) {  // client closed connection
            close(clnt_sock);
            delete this;  // assuming Reactor will clean up its handlers vector. Be careful with real code!
            return;
        }
        write(clnt_sock, buffer, str_len);
    }
};

class Acceptor : public EventHandler {
    int serv_sock;
    struct sockaddr_in serv_addr;

public:
    Acceptor(int port) {
        serv_sock = socket(PF_INET, SOCK_STREAM, 0);
        memset(&serv_addr, 0, sizeof(serv_addr));
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
        serv_addr.sin_port = htons(port);

        bind(serv_sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
        listen(serv_sock, 5);
    }

    int get_handle() {
        return serv_sock;
    }

    void handle_read() {
        struct sockaddr_in clnt_addr;
        socklen_t clnt_addr_size = sizeof(clnt_addr);
        int clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_addr, &clnt_addr_size);

        new EchoHandler(clnt_sock);  // Reactor should manage this in real code!
    }
};

class Reactor {
    std::vector<EventHandler*> handlers;
public:
    void register_handler(EventHandler* handler) {
        handlers.push_back(handler);
    }

    void remove_handler(EventHandler* handler) {
        handlers.erase(std::remove(handlers.begin(), handlers.end(), handler), handlers.end());
    }

    void handle_events() {
        while (true) {
            for (auto &handler : handlers) {
                handler->handle_read();
            }
        }
    }
};

int main() {
    Acceptor acceptor(12345);
    Reactor reactor;

    reactor.register_handler(&acceptor);
    reactor.handle_events();

    return 0;
}
```
