FROM php:8.4-fpm
RUN apt-get update && apt-get install -y --no-install-recommends \
        libpq-dev \
        libicu-dev \
        libzip-dev \
        git \
        unzip \
    &&  docker-php-ext-install  pdo_pgsql pcntl intl bcmath zip \
    &&  pecl install redis \
    && docker-php-ext-enable redis \
    && rm -rf /var/lib/apt/lists/*
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
ARG UID=1000
ARG GID=1000
RUN groupadd -g ${GID} app \
    && useradd -u ${UID} -g ${GID} -m -s /bin/bash app
USER app
WORKDIR /var/www/html