FROM haproxy:2.8@sha256:b748df80c4a9dbfd72d977f649318c3f447ac467d4674552d97a9e29272b1302

USER root
RUN apt-get update && apt-get install -y curl && apt-get clean

COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg