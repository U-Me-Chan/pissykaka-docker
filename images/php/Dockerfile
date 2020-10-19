FROM library/php:7.4-fpm
RUN docker-php-ext-install pdo_mysql
RUN apt update -yyq && apt install xz-utils zip unzip libzip-dev --no-install-recommends -yyq
RUN docker-php-ext-install zip
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
RUN php composer-setup.php --install-dir=/usr/local/bin/ --filename composer && rm -f composer-setup.php
