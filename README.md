# DockerPHP

##### Версия образа: ```8.1.0RC1-dev-laravel```
##### Ссылка на Docker HUB: [8.1.0RC1-dev-laravel](https://hub.docker.com/layers/173050329/on1kel/php/8.1.0RC1-dev-laravel/images/sha256-b52e3db3df07bfadb6e7880123541375a5d5378bad91641d9d1363843b7ca590?context=repo)

##### Версия PHP: ```8.1.0RC1```
##### Установленные пакеты:
 * libmemcached-dev
 * libzip-dev
 * libz-dev
 * libzip-dev
 * libpq-dev
 * libjpeg-dev
 * libpng-dev
 * libfreetype6-dev
 * libssl-dev
 * openssh-server
 * libmagickwand-dev
 * pkg-config
 * git
 * cron
 * nano
 * libxml2-dev
 * libreadline-dev
 * libgmp-dev
 * mariadb-client
 * unzip

##### Установленные расширения:
 * mongodb-1.11.0alpha1
 * soap
 * exif
 * pcntl
 * zip
 * pdo_mysql
 * pdo_pgsql
 * bcmath
 * intl
 * gmp
 * redis
 * imagick
 * gd
 * xdebug
 * memcached

#### Версия Composer: ```2.1.9 2021-10-05 09:47:38```

#### Для Laravel 8.* проекта в корне создать Dockerfile со следующим содержимым:
```
FROM on1kel/php:8.1.0RC1-dev-laravel

COPY . /var/www

RUN composer install --no-interaction --optimize-autoloader --no-dev
RUN composer update
RUN php artisan config:cache && \
    php artisan cache:clear && \
    php artisan clear-compiled && \
    php artisan view:cache && \
    php artisan view:clear && \
    php artisan route:cache

```

#### В .env добавить следующие переменные:
```
# --------------------- DOCKER --------------------- #
DOCKER_MONGO_PORT=27018
DOCKER_ELASTICSEARCH_PORT=9400
DOCKER_KIBANA_PORT=5601
DOCKER_REDIS_PORT=6380
DOCKER_APP_PORT=8081
DOCKER_NETWORKS_DRIVER=bridge
COMPOSE_PROJECT_NAME=cubekit.main.api
```

#### Для локального запуска использовать следующий docker-compose.yaml:
```yaml
version: '3.9'

networks:
  app_network:
    name: ${APP_NAME}.app.network
    driver: ${DOCKER_NETWORKS_DRIVER}
  redis_network:
    name: ${APP_NAME}.redis.network
    driver: ${DOCKER_NETWORKS_DRIVER}
  mongo_network:
    name: ${APP_NAME}.mongo.network
    driver: ${DOCKER_NETWORKS_DRIVER}
  elastic_network:
    name: ${APP_NAME}.elastic.network
    driver: ${DOCKER_NETWORKS_DRIVER}

services:
  app:
    build:
      context: .
      dockerfile: ./Dockerfile
    container_name: ${APP_NAME}.app
    command: ["php-fpm"]
    expose:
      - 9000
    volumes:
      - .:/var/www
    depends_on:
      - redis
      - mongo
      - elastic
    networks:
      - app_network
      - redis_network
      - mongo_network
      - elastic_network
    restart: always

  queue:
    build:
      context: .
      dockerfile: ./Dockerfile
    container_name: ${APP_NAME}.app.queue
    command: ["php", "/var/www/artisan", "queue:work", "--tries=3", "--timeout=90"]
    volumes:
      - .:/var/www
    depends_on:
      - app
      - redis
      - mongo
      - elastic
    networks:
      - app_network
      - redis_network
      - mongo_network
      - elastic_network
    restart: always

  scheduler:
    build:
      context: .
      dockerfile: ./Dockerfile
    container_name: ${APP_NAME}.app.scheduler
    command: ["php", "/var/www/artisan", "schedule:run"]
    volumes:
      - .:/var/www
    depends_on:
      - app
      - redis
      - mongo
      - elastic
    networks:
      - app_network
      - redis_network
      - mongo_network
      - elastic_network
    restart: always

  nginx:
    image: nginx
    container_name: ${APP_NAME}.nginx
    ports:
      - "${DOCKER_APP_PORT}:80"
    volumes:
      - .:/var/www
      - ./docker/data/nginx/conf.d/:/etc/nginx/conf.d/
      - ./docker/logs/nginx:/var/log/nginx
    depends_on:
      - app
    networks:
      - app_network

  redis:
    image: redis
    container_name: ${APP_NAME}.redis
    ports:
      - "${DOCKER_REDIS_PORT}:6379"
    expose:
      - 6379
    volumes:
      - ./docker/data/redis:/data/app
      - ./docker/logs/redis:/data/logs
    networks:
      - redis_network

  # Детали: https://hub.docker.com/_/mongo/
  mongo:
    image: mongo
    container_name: ${APP_NAME}.mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MD_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MD_PASSWORD}
    ports:
      - "${DOCKER_MONGO_PORT}:27017"
    expose:
      - 27017
    volumes:
      - ./docker/data/mongo/db:/data/db
      - ./docker/data/mongo/configdb:/data/configdb
      - ./docker/logs/mongo:/var/log/mongodb
    networks:
      - mongo_network

  elastic:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.5.2
    container_name: ${APP_NAME}.elastic
    environment:
      - xpack.security.enabled=false # for dev
      - discovery.type=single-node # for dev
    #      - node.name=elastic
    #      - cluster.name=es-docker-cluster
    #      - discovery.seed_hosts=es02,es03
    #      - cluster.initial_master_nodes=elastic
    #      - bootstrap.memory_lock=true
    #      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile: # for dev
        soft: 65536 # for dev
        hard: 65536 # for dev
    cap_add: # for dev
      - IPC_LOCK # for dev
    volumes:
      - ./docker/data/elastic:/usr/share/elasticsearch/data
    ports:
      - "${DOCKER_ELASTICSEARCH_PORT}:9200"
      - '9300:9300'
    expose:
      - 9200
    networks:
      - elastic_network

  kibana:
    image: kibana:7.5.2
    container_name: ${APP_NAME}.kibana
    volumes:
      - ./docker/data/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml
    expose:
      - '5601'
    ports:
      - "${DOCKER_KIBANA_PORT}:5601"
    restart: always
    networks:
      - elastic_network
    environment:
      - XPACK_GRAPH_ENABLED=true
      - TIMELION_ENABLED=true
      - ELASTICSEARCH_URL=http://elastic:9200
      - ELASTICSEARCH_HOSTS=http://elastic:9200
    depends_on:
      - elastic

```
