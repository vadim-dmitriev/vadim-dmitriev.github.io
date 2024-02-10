---
title: "Docker multi-stage build"
author: "Вадим Дмитриев"
description: "Разбираем многоэтапную сборку Docker образов с целью уменьшения их размера"
date: 2024-02-10T17:46:56+03:00
tags:
    - "docker"
    - "golang"
summary: "Для оптимизации процесса сборки и размера итогового образа у Docker есть возможность многоэтапной сборки образа. В этой статье рассмотрим основные аспекты работы с ней."

draft: false
---

{{< image src=images/docker-logo.png alt=docker-logo >}}

Для оптимизации процесса сборки и размера итогового образа у Docker, начиная с версии 17.05, появилась возможность многоэтапной сборки (multi-stage build). Используя такую сборку можно:
- Создать минимально возможный по размеру образ, который содержал бы в себе только исполняемый бинарный файл нашего приложения и не байта больше
- Сделать сборку быстрее за счет параллельного запуска нескольких этапов

## Введение

Для начала предлагаю разобрать как выглядит и из чего состоит Dockerfile с многоэтапной сборкой:

```Dockerfile
# Stage 'build'
FROM golang:1.19 AS build
WORKDIR /app
COPY . .
RUN go build -o binary .

# Stage 'run'
FROM scratch AS run
COPY --from=build /app/binary .
ENTRYPOINT ["/binary"]
```

При первом рассмотрении кажется, что перед нами два отдельных Dockerfile\`a идущих друг за другом. На самом деле каждая из команд `FROM` создает новую стадию. В нашем примере сборка будет состоять из двух стадий.

Слегка видоизменилась команда выбора базового образа: `FROM golang:1.19 AS build`. Благодаря `AS` мы теперь можем именовать стадии и обращаться к ним в дальнейшем. 

Идея многоэтапной сборки состоит в том, что стадии могут заимствовать результаты работы (артефакты) другой. В примере выше мы:
1. Создаем стадию с именем build на основе образа golang:1.19
2. Компилируем проект в бинарный файл binary
3. Создаем вторую стадию run на основе образа scratch (о нем чуть позже)
4. Копируем binary, собранный на стадии build, при помощи указания ключа `--from=build`
5. Запускаем binary во второй стадии. Заметьте, что на первой стадии у нас нет ENTRYPOINT

Подробнее хотел бы остановиться на некоторых особенностях работы с Docker multi-stage build:

- В общем случае мы можем не именовать стадии (не использовать `AS`). Тогда они будут пронумерованы числами начиная с нуля.

    <details>
        <summary>Пример стадий без явных именований</summary>

    ```Dockerfile
    # Stage '0'
    FROM golang:1.19
    WORKDIR /app
    COPY . .
    RUN go build -o binary .

    # Stage '1'
    FROM scratch
    COPY --from=0 /app/binary . # <---
    ENTRYPOINT ["/binary"]
    ```
    </details>

- Значением ключа `--from` может быть не только название стадии, но и любой базовый образ. Таким образом можно скопировать внутрь стадии файлы из любого базового образа но не быть основанным на нем.

    <details>
        <summary>Пример копирования из базового образа</summary>

    ```Dockerfile
    # Stage '0'
    FROM golang:1.15
    WORKDIR /app
    COPY . .
    RUN go build -o binary cmd/main.go

    # Stage '1'
    FROM scratch
    COPY --from=0 /app/binary .
    COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf # <---
    ENTRYPOINT ["/binary"]
    ```
    </details>

- В качестве базового образа можно использовать стадию созданную выше. В таком случае нем не придется явно копировать артефакты, все файлы и так уже будут внутри. Этот механизм пригодится нам дальше в разделе о параллельном запуске.

    <details>
        <summary>Пример наследования стадий</summary>

    ```Dockerfile
    # Stage 'base'
    FROM golang:1.15 AS base

    # Stage 'worker_1'
    FROM base AS worker_1
    RUN echo 'hello from worker 1'

    # Stage 'worker_2'
    FROM base AS worker_2
    RUN echo 'hello from worker 2'
    ```
    </details>

## Scratch

Давайте немного поговорим про scratch. Это особый базовый образ, я бы назвал его  его "базовым в квадрате", потому что на его основе созданы alpine, debian и другие образы, которые мы обычно берем за базовые. У scratch нет тегов и его не нужно пуллить - он поставляется с самом докером. Основная особенность этого образа заключается в том, что он абсолютно пуст. В нем нет ни файловой системы, ни стандартных unix команд, ни, собственно, командной оболочки sh.

Но все же я бы не рекомендовал использовать scratch в качестве базового образа для своих приложений в production, потмому что в случае возникновений проблем будет невозможно подключиться к исполняему контейнеру через `docker exec`.

Но давайте от слов к делу. Рассмотрим несколько применений Docker multi-stage build. 

## Минимальный образ приложения

Перейдем к созданию минимально возможного образа с вашим приложением. Для него нам необходимо использовать базовый образ scratch с применением двух стадий сборки.

```Dockerfile
# Stage 'builder'
FROM golang:1.15 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o binary ./cmd

# Stage 'run'
FROM scratch AS run
COPY --from=builder /app/binary .
ENTRYPOINT [ "/binary" ]
```

Получились следующие стадии:
1. builder, основанная на golang:1.15, собирает нам бинарник приложения
2. run, основанная на пустом scratch, копирует бинарь из первой стадии и запускает его

Для сравнения размера получившегося образа давайте напишем еще один Dockerfile, но в нем не будем использовать стадии, то есть все сделаем в одной, как обычно и делается.

```Dockerfile
FROM golang:1.15
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o binary ./cmd
ENTRYPOINT [ "/binary" ] 
```

Взглянув на оба этих Dockerfile\`а можно заметить, что в качестве базовых образов используются обычные `golang:1.15`, а не `golang:1.15-alpine`, который как раз часто и использут для минимизации размера. Чтобы закрыть этот момент я добавил образы, основанные на `alpine` в сравнене ниже.

```bash
$ docker images my_application
REPOSITORY          TAG                 IMAGE ID            SIZE
my_application      alpine_not_scratch  44c5502b7688        508MB
my_application      not_scratch         a85c00f641f5        1.05GB
my_application      scratch             a18b08859c96        11.8MB

$ docker container ls
CONTAINER ID        IMAGE               STATUS              NAMES
5aad258b7d79        44c5502b7688        Up 25 seconds       alpine_not_scratch
71f1fead35a2        a85c00f641f5        Up 11 seconds       not_scratch
6d3ea6027866        a18b08859c96        Up 40 seconds       scratch
```

В результате видим, что размеры образов отличаются на порядки, а приложения работают одинаково успешно!

## Образ фронтенд приложения с nginx

Бывают ситуации, когда при сборке одного функционала необходимо использовать более одной зависимости. Предположим мы хотим получить работающий контейнер с frontend приложением, который бы  мог отдавать статические файлы. Допустим для роутинга статики возьмем базовый образ nginx. Но перед запуском нам нужно скомпилировать наш frontend на Vue в готовую статику. Для этого необходим node.js. И тут уже становится непонятным, какой образ брать за базовый: nginx или node?

{{< image src=images/nginx_vs_node.jpg alt=nginx-vs-node >}}

Предположим мы решили сперва собрать фронт, а потом его отдавать, значит возьмем за базовый node. Давайте посмотрим на реузьтат:

```Dockerfile
# Stage 'builder'
FROM node:alpine
WORKDIR /app

# Install nginx
RUN apk add --no-cache nginx && mkdir /www
COPY nginx.conf /etc/nginx/nginx.conf

# Build frontend
ENV NODE_OPTIONS=--openssl-legacy-provider
COPY . .
RUN npm install
RUN npm run build

# Copy static
COPY ./dist/ /www/

EXPOSE 80
ENTRYPOINT ["nginx", "-g", "daemon off;"]
```

Более компактным решением будет использование двух этих образов последовательно. На первой стадии мы компилируем проект в статику, взяв за основу node, а во второй просто скопируем получившуюся директорию со статикой внутрь образа nginx. Помимо сокращения размера образа мы немного выигрываем и во времени сборки, как как нам не нужно выполнять `RUN apk add nginx`. Но только при первой сборке, потому что при дальнейших сборках Docker закеширует слой с этой командой.

```Dockerfile
# Stage 'builder'
FROM node:alpine AS builder
WORKDIR /app
# Build frontend
ENV NODE_OPTIONS=--openssl-legacy-provider
COPY . .
RUN npm install
RUN npm run build

# Stage 'nginx'
FROM nginx:alpine AS nginx
# Copy statics from previous stage
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
ENTRYPOINT ["nginx", "-g", "daemon off;"]
```

На готовых образах вновь видим разницу в размерах на порядок! Вывод - в ситуациях, когда для сборки образа необходимо использвать более одной зависимости, лучше использовать сразу оба образа этих зависимостей, но с применением многоэтапной сборки.

```bash
$ docker images
REPOSITORY          TAG                   IMAGE ID            SIZE
frontend            two_stage             0dd4c54519cf        47.9MB
frontend            one_stage             1ea220d752e9        337MB
```

## Параллельный запуск стадий

Если в `Dockerfile` последовательно указать две стадии, которые наследуются от одной и той же созданной ранее, то он начнет выполнять их параллельно. Получается, если в одном образе вам нужно собрать два приложения то можно сделать это в разных стадях. Но такая оптимизация сработает только если включена поддержка [BuildKit](https://docs.docker.com/build/buildkit/). Начиная с Docker версии 23.0 этот BuildKit уже включен.

```Dockerfile
# Stage 'base'
FROM golang:1.15 AS base
WORKDIR /app
COPY . .

# Stage 'client'
FROM base AS client
RUN go build -o client ./cmd/client

# Stage 'server'
FROM base AS server
RUN go build -o server ./cmd/server

FROM scratch
COPY --from=client /app/client .
COPY --from=server /app/server .
```

{{< image src=images/parallel_build.gif alt=parallel_build >}}
