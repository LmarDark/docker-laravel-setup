# docker-laravel-setup

## Tópicos
1
2
3
4
5

## O que é
Este repositório fornece uma configuração pronta para uso de um ambiente de desenvolvimento Docker para aplicações Laravel. Ele inclui contêineres pré-configurados para o servidor web, banco de dados e outras dependências necessárias, facilitando a inicialização e o gerenciamento de um ambiente Laravel local de forma rápida e eficiente.

## Configurações

### Setup para _Laravel / Vue / Inertia SSR_
```Dockerfile
FROM php:8.3-fpm

# Instale as dependências
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    nodejs \
    npm

# Limpa o cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Instalar extensões PHP
RUN apt-get update && apt-get install -y \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    libldap2-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
    mbstring exif pcntl bcmath gd ldap sockets

# Definr diretório raiz do contâiner
WORKDIR /var/www

# Copia todos os arquivos da pasta do Dockerfile para o contâiner
COPY . .

# Instalar dependências do Node.JS usadas no projeto
RUN npm install

# Copia as configurações personalizadas do PHP
COPY docker/php/custom.ini /usr/local/etc/php/conf.d/custom.ini

USER $user
```

```docker-compose.yml
services:
    app:
        build:
            context: .
            dockerfile: Dockerfile
        restart: unless-stopped
        working_dir: /var/www/
        command: ["/bin/sh", "/var/www/start.sh"]
        volumes:
            - ./:/var/www
        ports:
            - "5173:5173"
            - "13714:13714"
        networks:
            - laravel

    nginx:
        image: nginx:alpine
        restart: unless-stopped
        ports:
            - "80:80"
        volumes:
            - ./docker/nginx/:/etc/nginx/conf.d/
        networks:
            - laravel

networks:
    laravel:
        driver: bridge
```

```sh
#!/bin/bash
cd /var/www

# Instala Node.js e npm
apt-get update && apt-get install -y \
    nodejs \
    npm \ 
    php

# Baixa e instala o Composer
curl -sS https://getcomposer.org/installer -o composer-setup.php

# Verifica a integridade do instalador
HASH="$(curl -sS https://composer.github.io/installer.sig)"
if [ "$(php -r "echo hash_file('SHA384', 'composer-setup.php');")" != "$HASH" ]; then
    echo 'Installer corrupt' && unlink composer-setup.php
    exit 1
fi

php composer-setup.php --install-dir=/usr/local/bin --filename=composer
rm composer-setup.php

# Instala as dependências do projeto Laravel
composer install

# Limpa o cache do Laravel
php artisan config:clear
php artisan cache:clear

# Inicia o servidor PHP (em segundo plano)
php artisan serve --host=0.0.0.0 --port=9000 &

# Aguarda um momento para garantir que o servidor esteja iniciado
sleep 5

# Inicia o Vite (em primeiro plano)
npm run build &

wait
```

```custom.ini
[PHP]
post_max_size = 100M
upload_max_filesize = 100M
```

```nginx
server {
    listen 80;

    client_max_body_size 51g;
    client_body_buffer_size 512k;
    client_body_in_file_only clean;

    location / {
        proxy_pass http://app:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /@vite {
        proxy_pass http://app:5173; 
    }

    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
}
```
