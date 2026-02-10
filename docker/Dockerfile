FROM php:8.2-apache

# Install system dependencies and missing PHP extensions
RUN apt-get update && apt-get install -y \
    libzip-dev \
    libpng-dev \
    libonig-dev \
    libcurl4-gnutls-dev \
    libicu-dev \
    libxml2-dev \
    zip \
    unzip \
    git \
    curl \
    && docker-php-ext-install -j$(nproc) \
    pdo \
    pdo_mysql \
    mbstring \
    xml \
    xmlwriter \
    curl \
    bcmath \
    intl \
    zip \
    opcache \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Install Node.js for frontend assets
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
    apt-get install -y nodejs && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install and enable Xdebug for PHP debugging (commented out for Codespaces stability)
RUN pecl install xdebug \
    && docker-php-ext-enable xdebug

# Copy custom Xdebug configuration (commented out)
COPY docker/xdebug.ini /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini    

# Set working directory
WORKDIR /var/www/html

# Copy composer files first for better caching
COPY composer.json composer.lock* ./

# Clean apt cache to reduce image size
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install PHP dependencies
RUN composer install --optimize-autoloader --no-interaction || true

# Copy application files
COPY . .

# Set permissions
RUN mkdir -p /var/www/html/storage /var/www/html/bootstrap/cache && \
    chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache

# Create necessary Laravel directories
RUN mkdir -p /var/www/html/storage/framework/{sessions,views,cache} \
    /var/www/html/storage/logs \
    /var/www/html/bootstrap/cache

# Generate Passport OAuth keys with correct permissions
RUN php artisan passport:keys --force && \
    chmod 644 /var/www/html/storage/oauth-private.key /var/www/html/storage/oauth-public.key
    
# Configure Apache
RUN echo '<VirtualHost *:80>\n' \
    '    ServerName localhost\n' \
    '    DocumentRoot /var/www/html/public\n' \
    '\n' \
    '    <Directory /var/www/html/public>\n' \
    '        Options Indexes FollowSymLinks\n' \
    '        AllowOverride All\n' \
    '        Require all granted\n' \
    '    </Directory>\n' \
    '\n' \
    '    ErrorLog ${APACHE_LOG_DIR}/error.log\n' \
    '    CustomLog ${APACHE_LOG_DIR}/access.log combined\n' \
    '</VirtualHost>' > /etc/apache2/sites-available/000-default.conf

# Enable required Apache modules
RUN a2enmod rewrite headers

# Expose port 80
EXPOSE 80

# Ensure OAuth keys have correct permissions on startup
RUN echo '#!/bin/bash\n' \
    'if [ -f storage/oauth-private.key ]; then\n' \
    '    chmod 644 storage/oauth-private.key storage/oauth-public.key\n' \
    'fi\n' \
    'exec apache2-foreground' > /entrypoint.sh && \
    chmod +x /entrypoint.sh

CMD ["/entrypoint.sh"]
