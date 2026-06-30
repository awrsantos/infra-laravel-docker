# Usa a imagem oficial do php-fpm 8.5
FROM php:8.5-fpm

# Label
LABEL maintainer=""

# Instala todas as dependências do sistema
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive && apt-get install -y \
    libssl-dev \
    curl \
    libcurl4-gnutls-dev \
    libxml2-dev \
    libicu-dev \
    libmcrypt4 \
    libmemcached11 \
    openssl \
    libbz2-dev \
    libzip-dev \
    unzip \
    libonig-dev \
    libpq-dev \
    libxslt1-dev \
    libpng-dev \
    libpng16-16 \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    && rm -rf /var/lib/apt/lists/*

# Instala chaves e configura o repositório moderno da NodeSource (Node 20)
RUN apt-get update && apt-get install -y ca-certificates gnupg \
    && mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" | tee /etc/apt/sources.list.d/nodesource.list \
    && apt-get update && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

# Instala extensões nativas do PHP que não dependem de configurações complexas
RUN docker-php-ext-install \
    bcmath \
    bz2 \
    zip \
    mbstring \
    intl \
    pdo \
    pdo_pgsql \
    xml \
    xsl \
    sockets

# Configura e instala a extensão GD (para manipulação de imagens)
RUN docker-php-ext-configure gd --with-jpeg=/usr/include/ --with-freetype=/usr/include/ \
    && docker-php-ext-install gd

# Instala, habilita a extensão APCu via PECL e limpa o cache do instalador
RUN pecl install apcu-5.1.28 \
    && docker-php-ext-enable apcu \
    && rm -rf /tmp/pear

# Injeta as configurações necessárias sem precisar alterar o php.ini
# apc.enabled=1: ativa o cache em ambiente web
# apc.shm_size=32M: define o tamanho da memória compartilhada
# apc.enable_cli=1: ativa o APCu para a linha de comando
RUN echo "apc.enabled=1" >> $PHP_INI_DIR/conf.d/docker-php-ext-apcu.ini \
    && echo "apc.shm_size=32M" >> $PHP_INI_DIR/conf.d/docker-php-ext-apcu.ini \
    && echo "apc.enable_cli=1" >> $PHP_INI_DIR/conf.d/docker-php-ext-apcu.ini

# Copia o Composer mais recente diretamente na imagem
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Define a pasta padrão dentro do container
WORKDIR /var/www

# Garante que o usuário www-data seja o dono da pasta interna de trabalho de forma recursiva (-R)
RUN chown -R www-data:www-data /var/www

# Expõe a porta
EXPOSE 9000

# Comando para executar o php-fpm em primeiro plano
CMD ["php-fpm"]