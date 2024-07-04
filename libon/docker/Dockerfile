# freeswitch image
FROM debian:bookworm-20240408

ARG NEXUS_USER
ARG NEXUS_PASSWORD

RUN apt update && apt install -y gnupg2 wget lsb-release 

RUN echo "machine nexus.build.libon.com login $NEXUS_USER password $NEXUS_PASSWORD" > /etc/apt/auth.conf && chmod 600 /etc/apt/auth.conf
RUN echo "deb [trusted=yes] https://nexus.build.libon.com/repository/signalwire-bookworm bookworm main" > /etc/apt/sources.list.d/freeswitch-mirror.list

RUN apt-get update && apt-get install -y libfreetype6 liblua5.3-0 libodbc2 libopus0 libpcre3 libsndfile1 libsofia-sip-ua0 libspandsp3 libspeexdsp1 libtpl0 lua5.3 speex
COPY freeswitch*.tar.gz /tmp/
RUN tar -zxvf /tmp/freeswitch*.tar.gz -C /

COPY mod_prometheus.so /opt/freeswitch/lib/freeswitch/mod/mod_prometheus.so
