---
layout: post
title: Ajustando permissões para alterar arquivos durante o build de uma imagem Docker com usuário non-root
date: 2018-04-24
img: docker-lego-1280x850.jpg
tags: [docker, tip, fast-food tip]
author: lemuel
brief: Resolvendo problemas de permissão de escria em diretório durante o build de imagem Docker usando usuário non-root.
---

## Contexto

Fui criar uma imagem Docker e deparei com um detalhe de permissõs da instrução `COPY` quando usamos um usuário diferente do root (padrão) durante o build.

## Problema

O problema é com as permissões do diretório quando alguma instrução `RUN` escreve no destino do `COPY`.

Você provavelmente vai se deparar com esse problema quando estiver executando `composer install`, `dep ensure`, `npm install` ou qualquer outra tarefa que escreva no diretório criado com as permissões padrão.

### Replicando

- crie um diretório **docker-copy-test**
- crie o **Dockerfile**
- faça o build usando `docker build --tag "docker-copy-test" .`

```
# Dockerfile example
FROM alpine:latest

RUN adduser -D -h /home/arquivei -s /bin/ash arquivei

COPY . /home/arquivei/app

WORKDIR /home/arquivei/app
USER arquivei

RUN echo "foo" > bar
```

### Erro

```
Step 6/6 : RUN echo "foo" > bar
 ---> Running in 0d2add4b7050
/bin/sh: can't create bar: Permission denied
The command '/bin/sh -c echo "foo" > bar' returned a non-zero code: 1
```

## Solução

Uma solução é usar a flag `--chown` passando usuário e grupo que estiver em uso pela instrução `USER`.

Altere o **Dockerfile** e refaça a imagem:

```
# Dockerfile example
FROM alpine:latest

RUN adduser -D -h /home/arquivei -s /bin/ash arquivei

COPY --chown=arquivei:arquivei . /home/arquivei/app

WORKDIR /home/arquivei/app
USER arquivei

RUN echo "foo" > bar
```

Para mais detalhes leia a documentação: [Docker Reference#copy](https://docs.docker.com/engine/reference/builder/#copy).


[Imagem de capa](https://www.flickr.com/photos/134416355@N07/31775353301) por [kyohei ito](https://www.flickr.com/photos/134416355@N07/) sob [CC BY-SA 2.0](https://creativecommons.org/licenses/by-sa/2.0/).
