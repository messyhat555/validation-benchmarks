FROM php:7.4-apache@sha256:c9d7e608f73832673479770d66aacc8100011ec751d1905ff63fae3fe2e0ca6d

COPY . /var/www/html/

# Check if curl is already available and install it if needed
RUN php -m | grep -i curl || (echo "Installing curl support for PHP" && \
    apt-get update -o Acquire::AllowInsecureRepositories=true \
        -o Acquire::AllowDowngradeToInsecureRepositories=true || true && \
    apt-get install -y --allow-unauthenticated libcurl4-openssl-dev pkg-config && \
    docker-php-ext-install curl && \
    apt-get clean && rm -rf /var/lib/apt/lists/*)

EXPOSE 80
