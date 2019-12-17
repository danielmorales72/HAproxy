<h2>HAproxy for Nginx in Docker</h2>

The purpose of this is to setup the HAproxy that will load balance the traffic towards the Nginx containers.

Three networks are created:

* public - bridged network that is used to map the host to HAproxy (simulates public network)
* bacup, private two additional bridged networks that are simulating private networks. From the security point of view only HAproxy is able to send requests or is a kind of proxy.


***Basic configuration for the HAproxy container:***

    FROM haproxy:latest as proxy
    COPY ./haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg

    RUN apt-get update && apt-get install -y iputils-ping


Below the basic configuration file, in general the stracture is divided into 4 parts:

* _global_
* _defaults_
* _backend_ - Here we will define servers which will reply towards HTTP requests
* _frontend_ - This is our HAproxy itself, we need to define the IP address or map everything to the port :80. 

Eventually ACL may be used for what kind of requests we would like to load-balance

    global
      debug                                   

    defaults
      mode http                                # enable http mode which gives of layer 7 filtering
      timeout connect 5000ms                   # max time to wait for a connection attempt to a server to succeed
      timeout client 50000ms                   # max inactivity time on the client side
      timeout server 50000ms                   # max inactivity time on the server side

    backend legacy
      balance roundrobin
      mode http
      server server1 192.168.200.101:80 check   >> Here you define the first Docker Nginx Server, check as the name suggests     verifies connectivity
      server server2 192.168.200.102:80 check   >> Second Server
      server server3 192.168.254.101:80 check   >> Third Server

    frontend http
      bind *:80
      mode http                          
      default_backend legacy                   # set the default server for all request
  


Regular Nginx configuration for Dockerfile

    FROM nginx:latest as web

    WORKDIR /usr/share/nginx/html/
    RUN rm -f index.html
    EXPOSE 80

    COPY ./index.html /usr/share/nginx/html/

    ENTRYPOINT ["nginx"]
    CMD ["-g", "daemon off;"]

Index.html is very basic, used here only to provide the information which server replied to request


Docker Compose file

    ---

    version : "3.3"

    networks:
      public:
        ipam:
          driver: default
          config:
            - subnet: 192.168.100.0/24
      private:
        ipam:
          driver: default
          config:
            - subnet: 192.168.200.0/24
      backup:
        ipam:
          driver: default
          config:
            - subnet: 192.168.254.0/24

    services:
      primary:
        build: ./nginx
        container_name: nginx_primary
        ports:
          - "80"
        networks:
          private:
            ipv4_address: 192.168.200.101
      backup:
        build: ./backup
        container_name: nginx_backup
        ports:
          - "80"
        networks:
          private:
            ipv4_address: 192.168.200.102
      backup_az:
        build: ./backup2
        ports:
          - "80"
        container_name: nginx_backup_az
        networks:
          backup:
            ipv4_address: 192.168.254.101
            aliases: 
              - nginx_backup
      proxy: 
        build: ./proxy
        ports:
          - "8080:80"
        networks:
          public:
            ipv4_address: 192.168.100.100
          private:
             ipv4_address: 192.168.200.100
          backup:
             ipv4_address: 192.168.254.100


<h2> TESTS </h2>

During normal operation traffic is distributed equally over all Docker Nginx servers as per Round Robin algortihm
  
    TKOZIKOW-M-J22X:proxy tkozikow$ curl localhost:8080
    Hello World Engine1
    TKOZIKOW-M-J22X:proxy tkozikow$ curl localhost:8080
    Hello World Enigine2
    TKOZIKOW-M-J22X:proxy tkozikow$ curl localhost:8080
    Backup AZ
    TKOZIKOW-M-J22X:proxy tkozikow$ curl localhost:8080
    Hello World Engine1
    TKOZIKOW-M-J22X:proxy tkozikow$ 


__Corresponding logs from HAproxy__

    Attaching to nginx_primary, nginx_backup, nginx_backup_az, stack_proxy_1
    proxy_1      | [NOTICE] 350/085037 (1) : New worker #1 (7) forked
    nginx_primary | 192.168.200.100 - - [17/Dec/2019:08:50:53 +0000] "GET / HTTP/1.1" 200 20 "-" "curl/7.54.0" "-"
    nginx_backup | 192.168.200.100 - - [17/Dec/2019:08:50:54 +0000] "GET / HTTP/1.1" 200 21 "-" "curl/7.54.0" "-"
    nginx_backup_az | 192.168.254.100 - - [17/Dec/2019:08:50:55 +0000] "GET / HTTP/1.1" 200 10 "-" "curl/7.54.0" "-"
    nginx_primary | 192.168.200.100 - - [17/Dec/2019:08:50:56 +0000] "GET / HTTP/1.1" 200 20 "-" "curl/7.54.0" "-"
    nginx_backup | 192.168.200.100 - - [17/Dec/2019:08:50:56 +0000] "GET / HTTP/1.1" 200 21 "-" "curl/7.54.0" "-"
    nginx_backup_az | 192.168.254.100 - - [17/Dec/2019:08:50:57 +0000] "GET / HTTP/1.1" 200 10 "-" "curl/7.54.0" "-"
    nginx_primary | 192.168.200.100 - - [17/Dec/2019:09:13:35 +0000] "GET / HTTP/1.1" 200 20 "-" "curl/7.54.0" "-"
    nginx_backup | 192.168.200.100 - - [17/Dec/2019:09:13:38 +0000] "GET / HTTP/1.1" 200 21 "-" "curl/7.54.0" "-"
    nginx_backup_az | 192.168.254.100 - - [17/Dec/2019:09:13:39 +0000] "GET / HTTP/1.1" 200 10 "-" "curl/7.54.0" "-"
    nginx_primary | 192.168.200.100 - - [17/Dec/2019:09:13:41 +0000] "GET / HTTP/1.1" 200 20 "-" "curl/7.54.0" "-"

__After simulating the failure__

    TKOZIKOW-M-J22X:proxy tkozikow$ docker-compose stop primary
    Stop nginx_primary ... done

     proxy_1      | [WARNING] 350/091549 (7) : Server legacy/server1 is DOWN, reason: Layer4 timeout, check duration: 2004ms. 2 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.

Due to checks affected server is not considered to serve any requests anymore

    TKOZIKOW-M-J22X:proxy tkozikow$ curl localhost:8080
    Hello World Enigine2
    TKOZIKOW-M-J22X:proxy tkozikow$ curl localhost:8080
    Backup AZ
    TKOZIKOW-M-J22X:proxy tkozikow$ curl localhost:8080
    Hello World Enigine2


