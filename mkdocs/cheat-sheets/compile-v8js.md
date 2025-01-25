# Compiling V8Js for Debian Bookworm

To compile **V8Js** for PHP 7.3 using a Docker build, follow these key steps:

## Docker Build Overview

1. **Install V8 Engine**:
   The Dockerfile installs Google’s **V8 JavaScript Engine** version `9.9.115.9`:
   - Installs essential build dependencies like `build-essential`, `libtinfo5`, `ninja-build`, etc.
   - Clones the V8 source code and required depot tools from Google.
   - Builds the V8 engine and places the compiled shared libraries (`*.so`) and binaries in `/opt/v8/lib`, and headers in `/opt/v8/include`.

2. **Install PHP V8Js Extension**:
   - The **V8Js** PHP extension is cloned from GitHub, compiled, and installed.
   - It is configured to link with the V8 engine libraries located in `/opt/v8`.
   - The PHP extension `v8js.so` is installed in `/usr/local/lib/php/extensions/no-debug-non-zts-20180731`.
   - Finally, the extension is enabled with `docker-php-ext-enable v8js`.

## Post-build Deployment

If you need to deploy the **V8Js** extension to a server **outside the Docker container**, you must:

1. **Copy V8 Libraries**:
   Copy all the V8 shared libraries and resources from `/opt/v8` in the Docker container to the same location (`/opt/v8`) on the server.

2. **Copy PHP V8Js Extension**:
   Copy the `v8js.so` file from `/usr/local/lib/php/extensions/no-debug-non-zts-20180731` in the container to the same directory on the server.

3. **Ensure PHP Configuration**:
   Copy the PHP configuration file for V8Js (`/usr/local/etc/php/conf.d/docker-php-ext-v8js.ini`) from the Docker container to the same PHP configuration directory on the server.

## Key Files to Transfer from Docker to Server

- **V8 Libraries**: Copy `/opt/v8` (both libraries and headers) from the container to the server.
- **PHP V8Js Extension**: Copy `/usr/local/lib/php/extensions/no-debug-non-zts-20180731/v8js.so` from the container to the server.
- **PHP Configuration**: Copy `/usr/local/etc/php/conf.d/docker-php-ext-v8js.ini` to the server’s PHP configuration directory.

By ensuring these files are in place and paths are correctly set up on the server, your **PHP 7.3** installation will be able to use the **V8Js** extension.

```dockerfile
FROM php:7.3-fpm as BASE_PHP
ENV V8_VERSION=9.9.115.9

RUN apt-get update -y --fix-missing && apt-get upgrade -y;

# Install v8js (see https://github.com/phpv8/v8js/blob/php7/README.Linux.md)
RUN apt-get install -y --no-install-recommends \
    libtinfo5 libtinfo-dev \
    build-essential \
    curl \
    git \
    libglib2.0-dev \
    libxml2 \
    python \
    patchelf \
    ninja-build \
    && cd /tmp \
    \
    && git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git --progress --verbose \
    && export PATH="$PATH:/tmp/depot_tools" \
    \
    && fetch v8 \
    && cd v8 \
    && git checkout $V8_VERSION \
    && gclient sync \
    \
    && tools/dev/v8gen.py -vv x64.release -- is_component_build=true use_custom_libcxx=false

RUN export PATH="$PATH:/tmp/depot_tools" \
    && cd /tmp/v8 \
    && ninja -C out.gn/x64.release/ \
    && mkdir -p /opt/v8/lib && mkdir -p /opt/v8/include \
    && cp out.gn/x64.release/lib*.so out.gn/x64.release/*_blob.bin out.gn/x64.release/icudtl.dat /opt/v8/lib/ \
    && cp -R include/* /opt/v8/include/ \
    && apt-get install patchelf \
    && for A in /opt/v8/lib/*.so; do patchelf --set-rpath '$ORIGIN' $A;done

# Install php-v8js
RUN cd /tmp \
    && git clone https://github.com/phpv8/v8js.git \
    && cd v8js \
    && git checkout php7 \
    && phpize \
    && ./configure --with-v8js=/opt/v8 LDFLAGS="-lstdc++" CPPFLAGS="-DV8_COMPRESS_POINTERS" \
    && make \
    && make test \
    && make install

RUN ls -la /usr/local/etc/php/conf.d/ \
    && ls -la /usr/local/lib/php/extensions/

RUN docker-php-ext-enable v8js
FROM php:7.3-fpm

ARG VCS_REF
ARG BUILD_DATE
LABEL  org.label-schema.build-date=$BUILD_DATE \
        org.label-schema.name="PHP 7.3-FPM with V8JS" \
        org.label-schema.vcs-ref=$VCS_REF \
        org.label-schema.vcs-url="https://github.com/nkahoang/docker-v8js-php"

COPY --from=BASE_PHP /opt /opt
COPY --from=BASE_PHP /usr/local/etc/php/conf.d/docker-php-ext-v8js.ini /usr/local/etc/php/conf.d/
COPY --from=BASE_PHP /usr/local/lib/php/extensions/no-debug-non-zts-20180731 /usr/local/lib/php/extensions/no-debug-non-zts-20180731
RUN apt-get update && apt-get install -y \
      libfreetype6-dev \
      libjpeg62-turbo-dev \
      libpng-dev
RUN docker-php-ext-configure gd --with-freetype-dir=/usr --with-jpeg-dir=/usr --with-png-dir=/usr \
    && docker-php-ext-install -j$(nproc) gd
    
```