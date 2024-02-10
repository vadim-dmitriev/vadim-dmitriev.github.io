---
title: "Docker Multistage Build"
author: "Вадим Дмитриев"
description: "Разбираем многоэтапную сборку Docker образов с целью уменьшения их размера"
date: 2024-02-08T17:46:56+03:00
tags:
    - "docker"
    - "opts"

draft: false
---

![docker logo](./images/Docker_Logo.png)

Для оптимизации итоговых образов у Docker есть возможность многоэтапной сборки. Используя её можно добиться минимально возможного по размеру образа, который содержал бы в себе только исполняемый код.
**TODO: добавить инфу начиная с какой версии появилась такой тип сборки**

## Синтакс

**TODO: добавить отличия многоэтапной сборки**

## Scratch

Для создания минимально возможного образа нашего приложения за основу возьмем базовый образ scratch. Это особый образ, его можно назвать базовым  для базовых образов (базовый в квадрате). На его основе созданы alpine, debian и тд. У него нет тегов и его не нужно пуллить - он поставляется с самом докером.

```Dockerfile
FROM golang:1.19-alpine

WORKDIR $GOPATH/src/github.com/vadim-dmitriev/mirror-layout-api

COPY go.mod ./
COPY go.sum ./

COPY . .

RUN go build -o mirror-layout-api cmd/main.go

EXPOSE 8080
EXPOSE 8081

CMD ["./mirror-layout-api"]
```

```Dockerfile
FROM golang:1.19-alpine AS build

WORKDIR /app

COPY go.mod ./
COPY go.sum ./

COPY . .

RUN CGO_ENABLED=0 go build -o mirror-layout-api cmd/main.go


FROM scratch AS run

COPY --from=build /app/mirror-layout-api .

ENV SSL_CERT_DIR=.
COPY --from=build /etc/ssl/certs/* .

EXPOSE 8080
EXPOSE 8081

ENTRYPOINT ["/mirror-layout-api"]
```

## Пример 1

**TODO: образ golang+scratch**

## Пример 2

**TODO: образ vue+nginx**
