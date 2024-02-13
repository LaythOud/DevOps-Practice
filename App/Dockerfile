FROM php:8.2-apache

# Install any needed packages
RUN apt-get update && apt-get install -y \
    libzip-dev \
    unzip

# Install PHP extensions
RUN docker-php-ext-install pdo pdo_mysql zip

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Copy composer.lock and composer.json to the working directory
COPY composer.lock composer.json /var/www/html/

# Install Laravel dependencies
RUN composer install --no-scripts --no-autoloader

# Copy the rest of the application code
COPY . /var/www/html

# Generate the optimized autoloader
RUN composer dump-autoload --optimize

# Set the correct permissions
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache

COPY cicd/apache.conf /etc/apache2/sites-available/

RUN a2ensite apache
RUN a2dissite 000-default.conf