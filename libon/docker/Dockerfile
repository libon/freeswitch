# freeswitch image
FROM debian:trixie

ARG SIGNALWIRE_TOKEN

RUN apt update && apt install -y curl gnupg2 libshout3 lsb-release wget

RUN wget --http-user=signalwire --http-password=$SIGNALWIRE_TOKEN -O /usr/share/keyrings/signalwire-freeswitch-repo.gpg https://freeswitch.signalwire.com/repo/deb/debian-release/signalwire-freeswitch-repo.gpg
RUN echo "machine freeswitch.signalwire.com login signalwire password $SIGNALWIRE_TOKEN" > /etc/apt/auth.conf && chmod 600 /etc/apt/auth.conf
RUN echo "deb [signed-by=/usr/share/keyrings/signalwire-freeswitch-repo.gpg] https://freeswitch.signalwire.com/repo/deb/debian-release/ `lsb_release -sc` main" > /etc/apt/sources.list.d/freeswitch.list

RUN apt-get update && apt-get install -y libfreetype6 liblua5.3-0 libodbc2 libopus0 libsndfile1 libsofia-sip-ua0 libspandsp3 libspeexdsp1 libtpl0 lua5.3 speex \
                                         curl dnsutils gettext-base httpie jq lua-socket lua-json lua-sec net-tools ngrep procps sngrep tcpdump tree vim
COPY freeswitch*.tar.gz /tmp/
RUN tar -zxvf /tmp/freeswitch*.tar.gz -C /

COPY mod_prometheus.so /opt/freeswitch/lib/freeswitch/mod/mod_prometheus.so
