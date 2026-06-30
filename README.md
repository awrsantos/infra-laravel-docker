### Ambiente de Desenvolvimento Laravel Multi-Projeto (Docker)

Este repositório contém uma estrutura universal em Docker para desenvolvimento de múltiplas aplicações Laravel simultâneas utilizando Nginx, PHP-FPM 8.5 (com Node 20 e Composer integrados) e PostgreSQL.

* * *

### 📁 Estrutura de Pastas

    ├── nginx/
    │   ├── conf.d/          # Configurações (.conf) de cada projeto
    │   └── Dockerfile
    ├── php8fpm/
    │   └── Dockerfile       # Imagem PHP 8.5 + Node 20 + Composer
    ├── projects/            # Código-fonte dos seus projetos Laravel
    ├── pg_backups/          # Backups automáticos gerados ao fazer 'down' na stack
    └── docker-compose.yml


* * *

### 🚀 Primeiros Passos e Novo Projeto

### 1\. Configuração do Servidor e Domínio Local

1.  Copie o arquivo base em `./nginx/conf.d/default.conf` para criar a configuração do seu novo projeto (ex: `myproject.conf`) e ajuste as diretivas `server_name` e `root` apontando para a nova pasta.
2.  Abra o arquivo `hosts` do seu sistema operacional como administrador e aponte o domínio desejado para o IP local:
    
    *   **Windows:** `C:\Windows\System32\drivers\etc\hosts`
    *   **Linux/macOS:** `/etc/hosts`
    
        127.0.0.1   myproject.local


### 2\. Subindo a Stack pela Primeira Vez

Na raiz do repositório, execute o comando abaixo. A flag `--build` garante a montagem das imagens personalizadas do Nginx e do PHP a partir dos seus respectivos Dockerfiles:

bash

    docker compose up -d --build


### 3\. Criando o Projeto Laravel

Acesse o terminal interno do container PHP para gerar a estrutura do framework:

bash

    docker exec -it app-php-fpm bash


Dentro do container, execute o instalador do Laravel na pasta correspondente ao seu domínio:

bash

    composer create-project laravel/laravel myproject


### 4\. Permissões de Pasta e Instalação de Dependências

Ainda dentro do terminal do container, navegue até a pasta do projeto para alinhar as permissões do sistema de arquivos e baixar os pacotes do Node:

bash

    cd myproject
    chown -R www-data:www-data storage bootstrap/cache
    npm install
    exit


### 5\. Configuração do Banco de Dados (.env)

Acesse a ferramenta de gerenciamento do banco de dados (pgAdmin) em `http://localhost:5050` para criar o seu banco de dados ou utilize o [comando de criação rápida via terminal](#criar-banco-terminal):

*   **Host name/address no pgAdmin:** `postgres`
*   **Maintenance database:** `mydatabase`
*   **Username:** `admin`
*   **Password:** `admin`

Alinhe os dados de conexão dentro do arquivo `.env` do seu projeto Laravel criado na pasta `projects/`:

env

    DB_CONNECTION=pgsql
    DB_HOST=postgres
    DB_PORT=5432
    DB_DATABASE=mydatabase
    DB_USERNAME=admin
    DB_PASSWORD=admin


### 6\. Configuração do Vite (Frontend)

Abra o arquivo `projects/myproject/vite.config.js` na sua máquina local e adicione o bloco `server` dentro de `defineConfig` para permitir que o Docker exponha o frontend:

javascript

    server: {
        host: '0.0.0.0', // Permite conexões externas ao container
        port: 5173,      // Porta interna/externa do projeto
        watch: {
            usePolling: true, // Essencial para detectar mudanças de arquivos em volumes Docker no Windows/macOS
            ignored: ['**/storage/framework/views/**'], // Ignora views compiladas do Blade para evitar loops
        }
    },


*(Nota: Para múltiplos projetos rodando simultaneamente, basta abrir uma nova porta consecutiva no seu `docker-compose.yml`, como `5174:5174`, e refletir essa porta no respectivo arquivo `vite.config.js` do segundo projeto).*

* * *

### 📦 Passo a Passo: Importando um Projeto Já Pronto / Existente

Siga este fluxo para acoplar de forma segura um projeto que veio de fora ou de outro repositório para dentro da sua infraestrutura:

1.  **Mover os Arquivos:** Adicione a pasta do seu projeto pronto dentro do diretório `projects/` (ex: `projects/meu-projeto`).
2.  **Configurar o Nginx:** Crie uma nova configuração em `./nginx/conf.d/meu-projeto.conf`, ajustando o `server_name` e o `root` (apontando para `/var/www/meu-projeto/public`).
3.  **Reiniciar o Webserver:** Atualize o Nginx para ler o novo arquivo de configuração:
    
    bash
    
        docker compose restart nginx


4.  **Mapear o Domínio:** Abra o arquivo `hosts` do seu sistema operacional como administrador e adicione a nova URL local (ex: `127.0.0.1 meu-projeto.local`).
5.  **Ajustar o Vite:** Abra o arquivo `projects/meu-projeto/vite.config.js` e adicione o bloco `server` idêntico ao detalhado no Passo 6 da seção anterior.
6.  **Configurar o Ambiente e Dependências:** Como projetos existentes vêm limpos do Git (sem dependências e sem `.env`), execute a sequência abaixo na máquina real:
    *   **A. Criar o arquivo `.env` a partir do exemplo do projeto:**
        
        bash
        
            cp projects/meu-projeto/.env.example projects/meu-projeto/.env


    *   **B. Instalar as dependências do ecossistema PHP (Composer):**
        
        bash
        
            docker compose exec -w /var/www/meu-projeto php-fpm composer install


    *   **C. Gerar a chave criptográfica do Laravel:**
        
        bash
        
            docker compose exec -w /var/www/meu-projeto php-fpm php artisan key:generate


    *   **D. Ajustar o `.env`:** Abra o novo arquivo `.env` gerado e configure o nome da aplicação, a URL local e as credenciais do PostgreSQL da stack.
    *   **E. Corrigir permissões de escrita dentro do contêiner:**
        
        bash
        
            docker compose exec php-fpm chown -R www-data:www-data /var/www/meu-projeto/storage /var/www/meu-projeto/bootstrap/cache


    *   **F. Instalar os pacotes de frontend do Node:**
        
        bash
        
            docker compose exec -w /var/www/meu-projeto php-fpm npm install


    *   **G. Iniciar o Vite em segundo plano (Se o projeto estiver em desenvolvimento):**
        
        bash
        
            docker exec -d app-php-fpm sh -c "cd meu-projeto && npm run dev"


    *   **H. Compilar os assets para Produção (Se o projeto já estiver finalizado):**
        
        bash
        
            docker compose exec -w /var/www/meu-projeto php-fpm npm run build


    *   **I. Criar o banco de dados e rodar as Migrations/Seeds:** Utilize o comando de criação rápida via terminal se o banco ainda não existir e, em seguida, popule as tabelas:
        
        bash
        
            docker compose exec -w /var/www/meu-projeto php-fpm php artisan migrate --seed


* * *

### 💻 Fluxo Diário de Desenvolvimento

Ao iniciar o seu dia de trabalho, execute a sequência abaixo no terminal da sua máquina real para levantar o ambiente de forma produtiva (com o Vite operando em segundo plano):

1.  **Subir a Stack Docker:**
    
    bash
    
        docker compose up -d

    
2.  **Ligar o Vite em segundo plano (Servidor de assets ao vivo):**
    
    bash
    
        docker exec -d app-php-fpm sh -c "cd myproject && npm run dev"

    
3.  **Desligar o ambiente ao encerrar o dia:**
    Este comando desliga os containers e aciona a rotina inteligente que faz o backup automatizado de todos os seus bancos de dados salvando-os na pasta `./pg_backups` com retenção automática de 14 dias.
    
    bash
    
        docker compose down
    

* * *

### 🛠️ Comandos Úteis (Executados na máquina real)

### Gerenciamento de Imagens e Serviços

*   **Forçar o Docker a reconstruir todas as imagens da stack:**
    
    bash
    
        docker compose up -d --build


*   **Forçar a reconstrução de uma imagem específica (ex: php-fpm):**
    
    bash
    
        docker compose up -d --build php-fpm


*   **Reiniciar o servidor web Nginx:**
    
    bash
    
        docker compose restart nginx


### Ecossistema Laravel (Artisan e Composer)

*   **Rodar qualquer comando Artisan sem entrar no container:**
    
    bash
    
        docker compose exec -w /var/www/myproject php-fpm php artisan [comando]


*   **Rodar as Migrations do banco de dados:**
    
    bash
    
        docker compose exec -w /var/www/myproject php-fpm php artisan migrate


*   **Instalar novos pacotes PHP via Composer:**
    
    bash
    
        docker compose exec -w /var/www/myproject php-fpm composer require [nome-do-pacote]


### Gerenciamento do PostgreSQL via Linha de Comando

*   **Criar um novo banco de dados no Postgres diretamente pelo terminal:**
<a id="criar-banco-terminal"></a>
    
    bash
    
        docker compose exec postgres psql -U admin -d postgres -c "CREATE DATABASE nome_do_banco;"