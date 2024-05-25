FROM php:8.2-apache

# Install base dependencies
RUN apt-get update && apt-get install -y -q --no-install-recommends \
        apt-transport-https \
        build-essential \
        ca-certificates \
        curl \
        git \
        libssl-dev \
        wget \
        vim \
    && rm -rf /var/lib/apt/lists/*

# Install APACHE
RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libpng-dev \
    && docker-php-ext-install -j$(nproc) iconv \
    && docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd

COPY ./000-default.conf /etc/apache2/sites-available/

# Add user
ARG PUID=1000
ENV PUID ${PUID}
ARG PGID=1001
ENV PGID ${PGID}
ARG APP_USER=app_user
ENV APP_USER ${APP_USER}

RUN groupadd -g ${PGID} ${APP_USER} && \
    useradd -l -u ${PUID} -g ${APP_USER} -m ${APP_USER} && \
    usermod -p "*" ${APP_USER} -s /bin/bash

# Install Node
USER ${APP_USER}

ARG NODE_VERSION=16.13.2
ENV NODE_VERSION ${NODE_VERSION}
ARG NVM_DIR=/home/${APP_USER}/.nvm
ENV NVM_DIR ${NVM_DIR}

RUN mkdir -p $NVM_DIR && \
    curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash \
    && . ${NVM_DIR}/nvm.sh \
    && nvm install ${NODE_VERSION} \
    && nvm use ${NODE_VERSION} \
    && nvm alias ${NODE_VERSION} \
    && npm config set fetch-retries 2 \
    && npm config set fetch-retry-factor 10 \
    && npm config set fetch-retry-mintimeout 10000 \
    && npm config set fetch-retry-maxtimeout 60000 \
    && ln -s `npm bin --global` /home/${APP_USER}/.node-bin

RUN echo "" >> ~/.bashrc && \
    echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.bashrc && \
    echo '[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  # This loads nvm' >> ~/.bashrc

USER root

ENV PATH $PATH:/home/${APP_USER}/.node-bin

COPY ./.npmrc /root/.npmrc
COPY ./.npmrc /home/${APP_USER}/.npmrc

# Set project owner
ARG PROJECT_PATH=/var/www/html
ENV PROJECT_PATH ${PROJECT_PATH}
RUN chown -R ${PUID}:${PGID} ${PROJECT_PATH}

# Install Composer

USER root

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Add the composer.json
COPY ./composer.json /home/${APP_USER}/.composer/composer.json
COPY ./auth.json /home/${APP_USER}/.composer/auth.json
RUN chown -R ${PUID}:${PGID} /home/${APP_USER}/.composer

# Export composer vendor path
RUN echo "" >> ~/.bashrc && \
    echo 'export PATH="$HOME/.composer/vendor/bin:$PATH"' >> ~/.bashrc

# Update composer
ARG COMPOSER_VERSION=2
ENV COMPOSER_VERSION ${COMPOSER_VERSION}
RUN set -eux; \
      if [ "$COMPOSER_VERSION" = "1" ] || [ "$COMPOSER_VERSION" = "2" ]; then \
          composer self-update --${COMPOSER_VERSION}; \
      else \
          composer self-update ${COMPOSER_VERSION}; \
      fi

USER ${APP_USER}

# Check if global install need to be ran
ARG COMPOSER_GLOBAL_INSTALL=false
ENV COMPOSER_GLOBAL_INSTALL ${COMPOSER_GLOBAL_INSTALL}

RUN if [ ${COMPOSER_GLOBAL_INSTALL} = true ]; then \
    # run the install
    composer global install \
;fi

# Check if auth file is disabled
ARG COMPOSER_AUTH=false
ENV COMPOSER_AUTH ${COMPOSER_AUTH}

RUN if [ ${COMPOSER_AUTH} = false ]; then \
    # remove the file
    rm /home/${APP_USER}/.composer/auth.json \
;fi

ARG COMPOSER_REPO_PACKAGIST
ENV COMPOSER_REPO_PACKAGIST ${COMPOSER_REPO_PACKAGIST}

RUN if [ ${COMPOSER_REPO_PACKAGIST} ]; then \
    composer config -g repo.packagist composer ${COMPOSER_REPO_PACKAGIST} \
;fi

# Export composer vendor path
RUN echo "" >> ~/.bashrc && \
    echo 'export PATH="~/.composer/vendor/bin:$PATH"' >> ~/.bashrc

USER ${APP_USER}

RUN export COMPOSER_AUTH=

USER root

RUN apt update && apt install certbot python3-certbot-apache -y -q --no-install-recommends
RUN echo "ServerName poychamp.com" >> /etc/apache2/apache2.conf

