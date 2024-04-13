FROM quay.io/oauth2-proxy/oauth2-proxy:v7.6.0-alpine

USER root

# Copy trusted certificate authorities:
RUN mkdir -p /etc/ssl/certs
COPY ./certs/* /etc/ssl/certs/

# Copy custom login and error pages:
RUN mkdir -p /etc/oauth2proxy/www
COPY ./www/* /etc/oauth2proxy/www/
