FROM php:7.3-fpm-buster
LABEL maintainer="mail@yalei.wang"

ARG MODE=default
ARG SERVER_NAME=magento.test
ENV MAGENTO_RUN_MODE ${MODE}
ENV SERVER_NAME ${SERVER_NAME}
ENV MYSQL_HOST db
ENV MYSQL_DATABASE magento
ENV MYSQL_USER magento
ENV MYSQL_PASSWORD magento
ENV MAGENTO_ADMIN_FRONTNAME admin
ENV MAGENTO_ADMIN_FIRST_NAME wang
ENV MAGENTO_ADMIN_LAST_NAME yalei
ENV MAGENTO_ADMIN_EMAIL mail@yalei.wang
ENV MAGENTO_ADMIN_USER wyl
ENV MAGENTO_ADMIN_PASSWORD wangyl123
ENV MAGENTO_LOCALE zh_Hans_CN
ENV MAGENTO_CURRENCY CNY
ENV MAGENTO_TIMEZONE Asia/Hong_Kong
ENV ES_PORT elasticsearch
ENV ES_HOST 9200
ENV CRON off

RUN apt-get update && apt-get install -y \
  cron \
  git \
  gzip \
  libbz2-dev \
  libfreetype6-dev \
  libicu-dev \
  libjpeg62-turbo-dev \
  libmcrypt-dev \
  libpng-dev \
  libsodium-dev \
  libssh2-1-dev \
  libxslt1-dev \
  libzip-dev \
  lsof \
  default-mysql-client \
  vim \
  zip \
  procps

RUN docker-php-ext-configure \
  gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/

RUN docker-php-ext-install \
  bcmath \
  bz2 \
  calendar \
  exif \
  gd \
  gettext \
  intl \
  mbstring \
  mysqli \
  opcache \
  pcntl \
  pdo_mysql \
  soap \
  sockets \
  sodium \
  sysvmsg \
  sysvsem \
  sysvshm \
  xsl \
  zip

RUN cd /tmp \
  && curl -O https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz \
  && tar zxvf ioncube_loaders_lin_x86-64.tar.gz \
  && export PHP_VERSION=$(php -r "echo PHP_MAJOR_VERSION.'.'.PHP_MINOR_VERSION;") \
  && export PHP_EXT_DIR=$(php-config --extension-dir) \
  && cp "./ioncube/ioncube_loader_lin_${PHP_VERSION}.so" "${PHP_EXT_DIR}/ioncube.so" \
  && rm -rf ./ioncube \
  && rm ioncube_loaders_lin_x86-64.tar.gz \
  && docker-php-ext-enable ioncube

RUN pecl channel-update pecl.php.net \
  && pecl install xdebug

RUN if [ "$MODE" = "developer" ]; then docker-php-ext-enable xdebug; fi

## Replace next lines with below commented out version once issue is resolved
# https://github.com/php/pecl-networking-ssh2/pull/36
# https://bugs.php.net/bug.php?id=78560
RUN curl -o /tmp/ssh2-1.2.tgz https://pecl.php.net/get/ssh2 \
  && pear install /tmp/ssh2-1.2.tgz \
  && rm /tmp/ssh2-1.2.tgz \
  && docker-php-ext-enable ssh2
#RUN pecl install ssh2-1.2 \
#  && docker-php-ext-enable ssh2

RUN groupadd -g 1000 app \
  && useradd -g 1000 -u 1000 -d /var/www -s /bin/bash app

RUN apt-get install -y gnupg \
  && curl -sL https://deb.nodesource.com/setup_14.x | bash - \
  && apt-get install -y nodejs \
  && mkdir /var/www/.config /var/www/.npm \
  && chown app:app /var/www/.config /var/www/.npm \
  && npm install -g grunt-cli

RUN curl -sS https://getcomposer.org/installer | \
  php -- --1 --install-dir=/usr/local/bin --filename=composer

RUN mkdir /var/www/.composer-global \
  && composer init --working-dir=/var/www/.composer-global --no-interaction \
  && composer require --working-dir=/var/www/.composer-global mage2tv/magento-cache-clean:^1.0

# RUN curl -s https://packages.blackfire.io/gpg.key | apt-key add - \
#   && echo "deb http://packages.blackfire.io/debian any main" | tee /etc/apt/sources.list.d/blackfire.list \
#   && apt-get update \
#   && apt-get install blackfire-agent blackfire-php

# The update/cron.php file has been removed in Magento 2.4.0, if this file exists on your installation, it can be safely removed.
# Any reference to update/cron.php and bin/magento setup:cron:run should also be removed from the crontab
# RUN printf '* *\t* * *\tapp\t%s/usr/local/bin/php /var/www/html/update/cron.php\n' >> /etc/crontab \
#   && printf '* *\t* * *\tapp\t%s/usr/local/bin/php /var/www/html/bin/magento cron:run\n' >> /etc/crontab \
#   && printf '* *\t* * *\tapp\t%s/usr/local/bin/php /var/www/html/bin/magento setup:cron:run\n#\n' >> /etc/crontab
RUN mkdir /var/www/html/var/ \
  && touch /var/run/crond.pid /var/www/html/var/.setup_cronjob_status /var/www/html/var/.update_cronjob_status \
  && chown -R app:app /var/run/crond.pid /var/spool/cron /etc/cron*

COPY ./conf/www.conf /usr/local/etc/php-fpm.d/
COPY ./conf/php.ini /usr/local/etc/php/
COPY ./conf/php-fpm.conf /usr/local/etc/
COPY ./bin/ /usr/local/bin/

RUN if [ "$MODE" = "developer" ]; then \
  curl -sSLO https://github.com/mailhog/mhsendmail/releases/download/v0.2.0/mhsendmail_linux_amd64 \
  && chmod +x mhsendmail_linux_amd64 \
  && mv mhsendmail_linux_amd64 /usr/local/bin/mhsendmail \
  && echo "sendmail_path = \"/usr/local/bin/mhsendmail --smtp-addr=mailhog:1025\"" >> /usr/local/etc/php/php.ini; \
  fi

RUN chmod a+x /usr/local/bin/config-init && chmod a+s /usr/sbin/cron

COPY ./src/ /var/www/html/

RUN mkdir -p /etc/nginx/html /sock \
  && chown -R app:app /etc/nginx /var/www /usr/local/etc/php/conf.d /sock \
  && chmod u+x /var/www/html/bin/magento \
  && chmod -R u+w /var/www/html/

RUN composer update

CMD ["php-fpm", "-F"]
ENTRYPOINT ["/usr/local/bin/config-init"]

USER app:app

VOLUME /var/www

WORKDIR /var/www/html

EXPOSE 9001
