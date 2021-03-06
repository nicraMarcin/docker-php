source https://hub.docker.com/_/php/


# docker-php

Create a Dockerfile in your PHP project
```FROM php:7.2-cli
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
CMD [ "php", "./your-script.php" ]
```
Then, run the commands to build and run the Docker image:

```$ docker build -t my-php-app .
$ docker run -it --rm --name my-running-app my-php-app
```
Run a single PHP script
For many simple, single file projects, you may find it inconvenient to write a complete Dockerfile. In such cases, you can run a PHP script by using the PHP Docker image directly:

```$ docker run -it --rm --name my-running-script -v "$PWD":/usr/src/myapp -w /usr/src/myapp php:7.2-cli php your-script.php
```
Note that all variants of the php image contain the PHP cli.

With Apache
More commonly, you will probably want to run PHP in conjunction with Apache httpd. Conveniently, there's a version of the PHP container that's packaged with the Apache web server.

Create a Dockerfile in your PHP project
```FROM php:7.2-apache
COPY src/ /var/www/html/
```
Where src/ is the directory containing all your PHP code. Then, run the commands to build and run the Docker image:

```$ docker build -t my-php-app .
$ docker run -d --name my-running-app my-php-app
```
We recommend that you add a php.ini configuration file, see the "Configuration" section for details.

Without a Dockerfile
If you don't want to include a Dockerfile in your project, it is sufficient to do the following:

`$ docker run -d -p 80:80 --name my-apache-php-app -v "$PWD":/var/www/html php:7.2-apache`
Changing DocumentRoot
Some applications may wish to change the default DocumentRoot in Apache (away from /var/www/html). The following demonstrates one way to do so using an environment variable (which can then be modified at container runtime as well):

```FROM php:7.1-apache

ENV APACHE_DOCUMENT_ROOT /path/to/new/root

RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
RUN sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf
```
How to install more PHP extensions
Many extensions are already compiled into the image, so it's worth checking the output of php -m or php -i before going through the effort of compiling more.

We provide the helper scripts docker-php-ext-configure, docker-php-ext-install, and docker-php-ext-enable to more easily install PHP extensions.

In order to keep the images smaller, PHP's source is kept in a compressed tar file. To facilitate linking of PHP's source with any extension, we also provide the helper script docker-php-source to easily extract the tar or delete the extracted source. Note: if you do use docker-php-source to extract the source, be sure to delete it in the same layer of the docker image.
```
FROM php:7.2-apache
RUN docker-php-source extract \
    # do important things \
    && docker-php-source delete
```
PHP Core Extensions
For example, if you want to have a PHP-FPM image with iconv and gd extensions, you can inherit the base image that you like, and write your own Dockerfile like this:
```
FROM php:7.2-fpm
RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libpng-dev \
    && docker-php-ext-install -j$(nproc) iconv \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd
```
Remember, you must install dependencies for your extensions manually. If an extension needs custom configure arguments, you can use the docker-php-ext-configure script like this example. There is no need to run docker-php-source manually in this case, since that is handled by the configure and install scripts.

See "Dockerizing Compiled Software" for a description of the technique Tianon uses for determining the necessary build-time dependencies for any bit of software (which applies directly to compiling PHP extensions).

PECL extensions
Some extensions are not provided with the PHP source, but are instead available through PECL. To install a PECL extension, use pecl install to download and compile it, then use docker-php-ext-enable to enable it:
```
FROM php:7.2-fpm
RUN pecl install redis-4.0.1 \
    && pecl install xdebug-2.6.0 \
    && docker-php-ext-enable redis xdebug
FROM php:5.6-fpm
RUN apt-get update && apt-get install -y libmemcached-dev zlib1g-dev \
    && pecl install memcached-2.2.0 \
    && docker-php-ext-enable memcached
```    

It is strongly recommended that users use an explicit version number in their pecl install invocations to ensure proper PHP version compatibility (PECL does not check the PHP version compatiblity when choosing a version of the extension to install, but does when trying to install it).

For example, memcached-2.2.0 has no PHP version constraints (https://pecl.php.net/package/memcached/2.2.0), but memcached-3.0.4 requires PHP 7.0.0 or newer (https://pecl.php.net/package/memcached/3.0.4). When doing pecl install memcached (no specific version) on PHP 5.6, PECL will try to install the latest release and fail.

Beyond the compatibility issue, it's also a good practice to ensure you know when your dependencies receive updates and can control those updates directly.

Unlike PHP core extensions, PECL extensions should be installed in series to fail properly if something went wrong. Otherwise errors are just skipped by PECL.

For example, pecl install memcached-2.2.0 && pecl install redis-2.2.8 instead of pecl install memcached-2.2.0 redis-2.2.8. However, docker-php-ext-enable memcached redis is fine to be all in one command.

For running the Apache variants as an arbitrary user, there are several choices:

If your kernel is version 4.11 or newer, you can add --sysctl net.ipv4.ip_unprivileged_port_start=0 and then --user should work as it does for FPM.
If you adjust the Apache configuration to use an "unprivileged" port (greater than 1024 by default), then --user should work as it does for FPM regardless of kernel version.
Otherwise, setting APACHE_RUN_USER and/or APACHE_RUN_GROUP should have the desired effect (for example, -e APACHE_RUN_USER=daemon or -e APACHE_RUN_USER=#1000 -- see the Apache User directive documentation for details on the expected syntax).
"E: Package 'php-XXX' has no installation candidate"
As of docker-library/php#542, this image blocks the installation of Debian's PHP packages. There is some additional discussion of this change in docker-library/php#551 (comment), but the gist is that installing Debian's PHP packages in this image leads to two conflicting installations of PHP in a single image, which is almost certainly not the intended outcome.

For those broken by this change and looking for a workaround to apply in the meantime while a proper fix is developed, adding the following simple line to your Dockerfile should remove the block (with the strong caveat that this will allow the installation of a second installation of PHP, which is definitely not what you're looking for unless you really know what you're doing):

`RUN rm /etc/apt/preferences.d/no-debian-php`
The proper solution to this error is to either use FROM debian:XXX and install Debian's PHP packages directly, or to use docker-php-ext-install, pecl, and/or phpize to install the necessary additional extensions and utilities.

Configuration
This image ships with the default php.ini-development and php.ini-production configuration files.

It is strongly recommended to use the production config for images used in production environments!

The default config can be customized by copying configuration files into the `$PHP_INI_DIR
