FROM php:7.3-apache
LABEL maintainer="arthurpsmith@gmail.com"

# System dependencies
RUN set -eux; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        git \
        librsvg2-bin \
        imagemagick \
        # SyntaxHighlighting
        python3 \
	; \
    rm -rf /var/lib/apt/lists/*

# Install the PHP extensions we need
RUN set -eux; \
    \
    savedAptMark="$(apt-mark showmanual)"; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libicu-dev \
        ; \
    \
    docker-php-ext-install -j "$(nproc)" \
        intl \
        mbstring \
        mysqli \
        opcache \
    ; \
    pecl install apcu-5.1.18; \
    docker-php-ext-enable \
        apcu \
    ; \
    \
    apt-mark auto '.' > /dev/null; \
    apt-mark manual $savedAptMark; \
    ldd "$(php -r 'echo ini_get("extension_dir");')"/.so \
        | awk '/=>/ { print $3 }' \
        | sort -u \
        | xargs -r dpkg-query -S \
        | cut -d: -f1 \
        | sort -u \
        | xargs -rt apt-mark manual; \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    rm -rf /var/lib/apt/lists/*

# Enable Short URLs
RUN set -eux; \
    a2enmod rewrite \
    && { \
            echo '<Directory /var/www/html>'; \
            echo ' RewriteEngine On'; \
            echo ' RewriteCond %{REQUEST_FILENAME} !-f'; \
            echo ' RewriteCond %{REQUEST_FILENAME} !-d'; \
            echo ' RewriteRule ^ %{DOCUMENT_ROOT}/index.php [L]'; \
            echo '</Directory>'; \
        } > "$APACHE_CONFDIR/conf-available/short-url.conf" \
    && a2enconf short-url

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN { \
    echo 'opcache.memory_consumption=128'; \
    echo 'opcache.interned_strings_buffer=8'; \
    echo 'opcache.max_accelerated_files=4000'; \
    echo 'opcache.revalidate_freq=60'; \
    echo 'opcache.fast_shutdown=1'; \
    } > /usr/local/etc/php/conf.d/opcache-recommended.ini

# SQLite Directory Setup
RUN set -eux; \
    mkdir -p /var/www/data; \
    chown -R www-data:www-data /var/www/data
    
# MediaWiki setup - specify 1.35.0-wmf.38 version
RUN set -eux; \
    git clone https://gerrit.wikimedia.org/r/mediawiki/core.git /var/www/html

RUN cd /var/www/html \
    git checkout tags/1.35.1 \
    git submodule update --init \
    git submodule update --init vendor

RUN apt-get update; \
    apt-get install -q -y curl nodejs npm

RUN npm install npm@latest -g

RUN cd /var/www/html/extensions \
    git clone https://gerrit.wikimedia.org/r/mediawiki/extensions/UniversalLanguageSelector.git \
    git clone https://github.com/google/abstracttext.git /var/www/html/extensions/AbstractText \
    cd /var/www/html/extensions/AbstractText/eneyj \
    npm install \
    npm install uuid \
    mkdir config \
    mkdir config/runs \
    node src/scripts/labelMap.js \
    cp src/scripts/calibration-initial.json config/calibration.json \
    node src/scripts/measure.js --chatty \
    node src/scripts/calibrate.js

COPY DockerLocalSettings.php /var/www/html/DockerLocalSettings.php
COPY AdditionalLocalSettings.php /tmp/AdditionalLocalSettings.php
COPY admin_passwd /tmp/admin_passwd

RUN cd /var/www/html \
    php maintenance/install.php --server http://localhost:8081 --dbtype sqlite --dbpath /var/www/data --scriptpath "" --passfile /tmp/ admin_passwd "Lambda" "admin"

RUN cat /tmp/AdditionalLocalSettings.php >> /var/www/html/LocalSettings.php

RUN cd /var/www/html \
    php maintenance/update.php

RUN cd /var/www/html \
    php maintenance/importTextFiles.php -s "Import data" --prefix "M:" --overwrite extensions/AbstractText/eneyj/data/Z*

RUN chown -R www-data /var/www/data