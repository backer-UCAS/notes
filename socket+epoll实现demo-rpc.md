服务端代码
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define MAX_EVENTS 10
#define BUFFER_SIZE 1024
#define PORT 5000

typedef struct {
    int fd;
    char buffer[BUFFER_SIZE];
    int buffer_len;
} client_t;

int add(int a, int b) {
    return a + b;
}

int subtract(int a, int b) {
    return a - b;
}

void handle_request(client_t *client) {
    char function[BUFFER_SIZE];
    int a, b;
    sscanf(client->buffer, "%s %d %d", function, &a, &b);
    int result;

    if (strcmp(function, "add") == 0) {
        result = add(a, b);
    } else if (strcmp(function, "subtract") == 0) {
        result = subtract(a, b);
    } else {
        snprintf(client->buffer, BUFFER_SIZE, "Unknown function\n");
        client->buffer_len = strlen(client->buffer);
        return;
    }

    snprintf(client->buffer, BUFFER_SIZE, "Result: %d\n", result);
    client->buffer_len = strlen(client->buffer);
}

int main() {
    int server_fd, epoll_fd;
    struct sockaddr_in server_addr;
    struct epoll_event ev, events[MAX_EVENTS];

    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        perror("bind");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    if (listen(server_fd, 5) == -1) {
        perror("listen");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) {
        perror("epoll_create1");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    ev.events = EPOLLIN;
    ev.data.fd = server_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev) == -1) {
        perror("epoll_ctl: server_fd");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    while (1) {
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        if (nfds == -1) {
            perror("epoll_wait");
            close(server_fd);
            exit(EXIT_FAILURE);
        }

        for (int n = 0; n < nfds; ++n) {
            if (events[n].data.fd == server_fd) {
                struct sockaddr_in client_addr;
                socklen_t client_addr_len = sizeof(client_addr);
                int client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_addr_len);
                if (client_fd == -1) {
                    perror("accept");
                    continue;
                }

                client_t *client = malloc(sizeof(client_t));
                client->fd = client_fd;
                client->buffer_len = 0;

                ev.events = EPOLLIN;
                ev.data.ptr = client;
                if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev) == -1) {
                    perror("epoll_ctl: client_fd");
                    close(client_fd);
                    free(client);
                    continue;
                }
            } else {
                client_t *client = (client_t*)events[n].data.ptr;

                if (events[n].events & EPOLLIN) {
                    int nread = read(client->fd, client->buffer, BUFFER_SIZE);
                    if (nread <= 0) {
                        if (epoll_ctl(epoll_fd, EPOLL_CTL_DEL, client->fd, NULL) == -1) {
                            perror("epoll_ctl: EPOLL_CTL_DEL");
                        }
                        close(client->fd);
                        free(client);
                    } else {
                        client->buffer[nread] = '\0';
                        handle_request(client);

                        ev.events = EPOLLOUT;
                        ev.data.ptr = client;
                        if (epoll_ctl(epoll_fd, EPOLL_CTL_MOD, client->fd, &ev) == -1) {
                            perror("epoll_ctl: EPOLL_CTL_MOD");
                            close(client->fd);
                            free(client);
                        }
                    }
                } else if (events[n].events & EPOLLOUT) {
                    int nwrite = write(client->fd, client->buffer, client->buffer_len);
                    if (nwrite == -1) {
                        perror("write");
                        if (epoll_ctl(epoll_fd, EPOLL_CTL_DEL, client->fd, NULL) == -1) {
                            perror("epoll_ctl: EPOLL_CTL_DEL");
                        }
                        close(client->fd);
                        free(client);
                    } else {
                        ev.events = EPOLLIN;
                        ev.data.ptr = client;
                        if (epoll_ctl(epoll_fd, EPOLL_CTL_MOD, client->fd, &ev) == -1) {
                            perror("epoll_ctl: EPOLL_CTL_MOD");
                            close(client->fd);
                            free(client);
                        }
                    }
                }
            }
        }
    }

    close(server_fd);
    return 0;
}
```

客户端代码
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUFFER_SIZE 1024
#define SERVER_IP "127.0.0.1"
#define SERVER_PORT 5000

int main() {
    int sockfd;
    struct sockaddr_in server_addr;
    char buffer[BUFFER_SIZE];

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    inet_pton(AF_INET, SERVER_IP, &server_addr.sin_addr);

    if (connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        perror("connect");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    // 向服务器发送请求
    snprintf(buffer, BUFFER_SIZE, "add 3 4");
    write(sockfd, buffer, strlen(buffer));

    // 接收服务器响应
    int nread = read(sockfd, buffer, BUFFER_SIZE);
    if (nread > 0) {
        buffer[nread] = '\0';
        printf("Received: %s", buffer);
    }

    close(sockfd);
    return 0;
}
```
