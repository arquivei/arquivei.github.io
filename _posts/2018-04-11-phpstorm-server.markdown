---
layout: post
title: Como usar o XDebug dentro de um container Docker
date: 2018-03-29
categories: blog
img: postman-start.png
tags: [Postman, Testing API, QA, Test]
author: edison_jr
brief: Como testar uma API Rest utilizando Postman + Newman
---

### Introdução

Quando escuto sobre debug em PHP:

"É muito dificil de debugar"
"Preciso escrever código de debug utilizando funçoes como var_dump() e die()."

Entre outras coisas. Mas não precisa ser assim.

Então vamos debugar em PHP sem a necessidade de escrever códigos como var_dump, dump, dd e die utilizando Docker.

### Docker install

Antes de mais nada, precisamos ter o Docker instalado no PC. Para instalar o docker vá para a documentação em (https://docs.docker.com/engine/installation/) e siga os passos.

### Phpdocker.io

Precisamos criar um conteiner do Docker com php-fpm e a extensão do xdebug instalada. Para isso podemos utilizar o phpdocker.io que cria automaticamente para nós.

Na verdade é bem simples, acesse a página do phpdocker.io ( https://phpdocker.io/generator ) e escolha as opções desejadas. As configurações são categorizadas em Global, PHP, Database e Zero-config.

Na aba global você precisa escolher apenas o nome do projeto e a porta, para o nome do projeto utilize "xdebug-docker", por exemplo, em application type, podemos utilizar o default (Generic: Zend, Laravel, Lumen...), para a porta padrão, pode-se utilizar :80, ou qualquer outra que quiser, nesse projeto é utilizada a porta :8200. E para o max upload size, você pode manter o padrão também (100MB).

Para a configuração do PHP você pode utilizar a versão 7.1, não há necessidade de instalar o Git e selecione a extensão do XDebug.

Não é necessária nenhuma configuração de database ou zero-config para esse projeto, podendo mante-los com as opções padrão.

### Dockerfile and docker-compose.yml

Precisamos configurar o arquivo docker-compose.yml e o Dockerfile, por hora podemos abrir o /phpdocker.io/php-fpm/Dockerfile em qualquer editor.

```
FROM phpdockerio/php71-fpm:latest
WORKDIR "/application"

# Install selected extensions and other stuff
RUN apt-get update \
    && apt-get -y --no-install-recommends install  php-xdebug \
    && apt-get clean; rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/*
```

Precisamos colocar as configurações do xdebug nesse arquivo. Lembre-se de adicionar um barra invertida (\) após doc/* na ultima linha antes de adicionar as linhas abaixo:

```
&& echo "zend_extension=/usr/lib/php/20160303/xdebug.so" > /etc/php/7.1/mods-available/xdebug.ini \
    && echo "xdebug.remote_enable=on" >> /etc/php/7.1/mods-available/xdebug.ini \
    && echo "xdebug.remote_handler=dbgp" >> /etc/php/7.1/mods-available/xdebug.ini \
    && echo "xdebug.remote_port=9000" >> /etc/php/7.1/mods-available/xdebug.ini \
    && echo "xdebug.remote_autostart=on" >> /etc/php/7.1/mods-available/xdebug.ini \
    && echo "xdebug.remote_connect_back=0" >> /etc/php/7.1/mods-available/xdebug.ini \
    && echo "xdebug.idekey=docker" >> /etc/php/7.1/mods-available/xdebug.ini
```

Essas linhas adicionam algumas configurações para o XDebug no php.ini, vamos ver o que cada configuração faz?

- remote_enable: Se ativa, tenta conectar com um Client, que você pode configurar usando remote_host e remote_port. Se a conexão falhar, essa opção é automaticamente desabilitada.

- remote_handler: seleciona o output do debugger, após a versão 2 a opção precisa ser DBPG, caso deseje saber mais sobre o common debugger protocol veja https://xdebug.org/docs-dbgp.php().

- remote_port: configura a porta de conexão do xdebug, o default é 9000.

- remote_autostart: Se true, o XDebug tentará inicializar e conectar com o host remoto.

- remote_connect_back: Se true, essa opção ignora o remote_host e vai tentar se conectar com o Client usando o IP de origem da requisição utilizando os cabeçalhos ($\_SERVER['HTTP_X_FORWARDED_FOR'] e $\_SERVER['REMOTE_ADDR']).

- idekey: Controla qual IDEKey o XDebug deve passar para o handler do DBGp.

Se você quiser saber mais sobre a configuração do XDebug, veja
https://xdebug.org/docs/all_setting.

Construa seu container:

```
    docker-compose up --build
```

Depois de construir o container, edite o arquivo docker-compose.yml, configurando o ambiente para o serviço php-fpm.

Configuramos o xdebug utilizando a chave (XDEBUG_CONFIG), precisamos então configurar o remote_host que vai receber como valor o endereço de IP da rede do Docker.

E para o PHPStorm utilizaremos a chave (PHP_IDE_CONFIG), precisamos configurar uma variavél de ambiente "serverName" que será a conexão que a IDE vai escutar o xdebug.

Se ainda existirem dúvidas sobre como você pode obter o IP do seu docker veja (https://linux.die.net/man/8/ifconfig) se for Linux ou (https://www.microsoft.com/resources/documentation/windows/xp/all/proddocs/en-us/ipconfig.mspx) se Windows.

Se não for utilizar PHPStorm, não há necessidade de utilizar PHP_IDE_CONFIG.

```
environment:
        XDEBUG_CONFIG: remote_host=172.17.0.1 #docker network ip.
        PHP_IDE_CONFIG: "serverName=xdebug-docker" #phpstorm variavel de ambiente com o nome do server configurado.
```

Reinicie seu container.

```
    docker-compose down
    docker-compose up -d
```

Nesse tutorial, você pode usar Microsoft Visual Studio Code ou Jetbrains PHPStorm.

Escolha seu editor favorito!

### Visual Studio Code

No VSC, clique em debug na barra de menu ou use a tecla de atalho F5, e escolha a opção PHP. O Editor vai criar um arquivo /.vs/launch.json, abra-o e coloque as seguintes configurações:

```
    "serverSourceRoot": "/application/public",
    "localSourceRoot": "${workspaceRoot}/public"
```

Crie um arquivo de hello world em /public/index.php:

```
<?php
$hello = "Hello World";
echo $hello;
```
Para adicionar um break point no Visual Code, clique com o botão direito e selecione a opção "Add Column Breakpoint" ou clique na coluna a esquerda antes do número da linha.


![alt-text]({{site.baseurl}}/assets/img/posts/image_vsc_debug.png)
Finalmente clique no icone de play verde "Listen for XDebug, no canto superior esquerdo e acesse http://127.0.0.1:8200 no seu navegador - use a porta que você configurou anteriormente - e pronto.

### PHPStorm

No PHPStorm vá para File -> Settings -> Languages & Frameworks -> PHP -> Server. E crie um servidor com o mesmo nome que você configurou no docker-compose.yml. Configure também seus path mappings.

![alt-text]({{site.baseurl}}/assets/img/posts/image_phpstorm_server.png)
Crie um arquivo de hello world em /public/index.php:

```
<?php
$hello = "Hello World";
echo $hello;
```

Para iniciar o debug vá até Run -> Start listening for PHP debug connections e adicione um breakpoint no PHPStorm clicando na coluna lateral antes do número da linha.

Execute no navegador http://127.0.0.1:8200 (ou utilize a porta que você escolheu anteriormente).

E pronto!

### Conclusão

Isso é tudo pessoal, agora sabemos como utilizar a extensão do PHP xdebug dentro de um container do docker.

Até mais.
