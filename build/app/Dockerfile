FROM node:20-alpine AS built_assets

ARG WORKPATH="/srv/app"

# copy project contents
ADD ./core "${WORKPATH}"

WORKDIR "${WORKPATH}"

# install packages
RUN npm install \
    # build
    && npm run build \
    # remove node modules
    && rm -rf ./node_modules


FROM php:8.2-fpm-bookworm

ARG BUILD_BASEPATH="./server/build/app"
ARG WORKPATH="/srv/app"

RUN apt update -y && apt install -y \
    # quick dev debug packages
    tree vim \
    # build packages
    git curl wget gnupg zip unzip build-essential make cmake

# install bento toolset for mp4 media inspection and transformation
ARG BENTO_SOURCE="https://github.com/axiomatic-systems/Bento4.git"
WORKDIR "/tmp/bento"

RUN git clone --depth=1 $BENTO_SOURCE . \
    && mkdir ./cmakebuild && cd ./cmakebuild \
    && cmake -DCMAKE_BUILD_TYPE=Release .. \
    && make \
    && mv mp4* /usr/local/bin

# install ffmpeg and ffmpegthumbnailer
RUN apt install -y ffmpeg ffmpegthumbnailer

# install and enable postgres extension
RUN apt update -y \
    && apt install -y libpq-dev postgresql-client-15 \
    && docker-php-ext-install pdo pdo_pgsql

# install and enable redis extension
RUN pecl install redis-6.0.1 \
    && docker-php-ext-enable redis

# install and enable imagick extension
RUN apt install -y libmagickwand-dev \
    && pecl install imagick \
    && docker-php-ext-enable imagick

# install and enable zip extension
RUN apt install -y libzip-dev \
    && docker-php-ext-install zip

# config and install php process controll extension
RUN docker-php-ext-configure pcntl --enable-pcntl \
    && docker-php-ext-install pcntl

# php config
WORKDIR "${PHP_INI_DIR}"

ADD "${BUILD_BASEPATH}/etc/php/conf.ini" "./php.ini"

# php-fpm config
WORKDIR "/usr/local/etc/php-fpm.d"

RUN rm -rf ./*
ADD "${BUILD_BASEPATH}/etc/php-fpm/www.conf" "./www.conf"
ADD "${BUILD_BASEPATH}/etc/php-fpm/zz-docker.conf" "./zz-docker.conf"


# create missing folders
ARG MEDIA_STORE="/data"

# set data folder for objects and thumb
RUN mkdir -pm 0770 "${MEDIA_STORE}" "${WORKPATH}" \
    # update ownership
    && chown www-data:www-data "${MEDIA_STORE}" "${WORKPATH}"

# copy project from source to dest
ADD  ./core "${WORKPATH}"

# switch to workdir
WORKDIR "${WORKPATH}"

# remove assets from public folder
RUN find ./public -maxdepth 1 ! -path ./public ! -ipath '*.php' | xargs -I % -- rm -rf %

# install server side packages via composer
RUN curl -o- https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
    # install server dependencies packages
    && composer install


ARG APP_KEY="--"
ARG DB_DATABASE="lab_core_db"
ARG DB_USERNAME="root"
ARG DB_PASSWORD="passwd"

ENV APP_KEY="${APP_KEY}"
ENV DB_DATABASE="${DB_DATABASE}"
ENV DB_USERNAME="${DB_USERNAME}"
ENV DB_PASSWORD="${DB_PASSWORD}"

