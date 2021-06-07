FROM nginx:1.18-alpine
LABEL maintainer="mail@yalei.wang"

ARG MODE=default
ARG SERVER_NAME=magento.test
ENV MAGENTO_RUN_MODE ${MODE}
ENV SERVER_NAME ${SERVER_NAME}
ENV UPLOAD_MAX_FILESIZE 32M
ENV FPM_HOST phpfpm
ENV FPM_PORT 9000
ENV CRON_HOST phpcron
ENV CRON_PORT 9000
ENV MAGENTO_ROOT /var/www/html

RUN addgroup -g 1000 app \
 && adduser -G app -u 1000 -h /var/www -s /bin/bash -S app
RUN touch /var/run/nginx.pid
RUN mkdir /sock

RUN apk add --no-cache \
  curl \
  nss-tools

COPY ./certs/ /etc/nginx/certs/
RUN if [ ! -f /etc/nginx/certs/nginx.key ] || [ ! -f /etc/nginx/certs/nginx.crt ]; then \
  cd /usr/local/bin/ \
  && curl -L https://github.com/FiloSottile/mkcert/releases/download/v1.4.1/mkcert-v1.4.1-linux-amd64 -o mkcert \
  && chmod +x mkcert \
  && mkcert -install \
  && mkcert -key-file nginx.key -cert-file nginx.crt ${SERVER_NAME} \
  && chown app:app nginx.key nginx.crt \
  && mv nginx.key nginx.crt /etc/nginx/certs/; \
  fi

COPY ./conf/nginx.conf /etc/nginx/
COPY ./conf/default.conf /etc/nginx/conf.d/

RUN mkdir -p /etc/nginx/html /var/www/html \
  && chown -R app:app /etc/nginx /var/www /var/cache/nginx /var/run/nginx.pid /sock

COPY ./bin/ /usr/local/bin/
RUN chmod +x /usr/local/bin/config-init
CMD ["nginx", "-g", "daemon off;"]
ENTRYPOINT ["/usr/local/bin/config-init"]

EXPOSE 8443

USER app:app

VOLUME /var/www

WORKDIR /var/www/html
