FROM haproxy:latest as proxy
COPY ./haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg

RUN apt-get update && apt-get install -y iputils-ping
