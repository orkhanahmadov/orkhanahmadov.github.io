---
layout: post
title: Using Imagick extension with PHP 8.3 on Docker
---

If you use the `imagick` extension on PHP 8.2 or earlier, it is quite straightforward to install on Docker.

```Dockerfile
RUN pecl install imagick && \
    docker-php-ext-enable imagick
```

But currently, and for some time the PECL version of the extension is broken on PHP 8.3.

<!--more-->

Instead, as a workaround, you can build the extension from official repository's `master` branch:

```Dockerfile
RUN apk add git --update --no-cache && \
    git clone https://github.com/Imagick/imagick.git --depth 1 /tmp/imagick && \
    cd /tmp/imagick && \
    git fetch origin master && \
    git switch master && \
    cd /tmp/imagick && \
    phpize && \
    ./configure && \
    make && \
    make install && \
    apk del git && \
    docker-php-ext-enable imagick
```

The `master` version on the `imagick` repository is compatible with PHP 8.3.
However, the PECL version is not updated yet.
Hopefully, it will get updated soon and we can switch back to the PECL version, instead of having this workaround.
