# Visão geral

O blog da engenharia do Arquivei usa a tecnologia Jekyll e tem como propósito usar o github para hospedar todas as publicações do blog. Além disso, esperamos que os autores se identifiquem para que a comunidade possa ver os seus feitos.

# Rodar localmente

Para evitar instalar qualquer dependência na máquina usaremos o **docker-compose** para rodar o jekyll localmente já que este faz parte da nossa stack. Para facilitar o desenvolvimento nós usaremos uma imagem pronto do docker hub e o seu uso é demasiadamente simples.

Para iniciar o serviço basta rodar: `docker-compose run --service-ports site jekyll s`e acessar a porta 4000. Esta também é configurável para o desenvolvimento do blog.

Você não precisa reinicar o serviço para cada modificação apenas quando os arquivos de configurações forem modificados como `_config.yml`.

ref: https://github.com/envygeeks/jekyll-docker

# Primeiro post

## Autor(imagem)

O primeiro passo é adicionar vocẽ ao grupo de autores. O primeiro passo é adicionar uma foto sua no diretório com as fotos dos autores em `assets/img/authors`. É indiferente o nome que você der para o arquivo, mas tente colocar um nome mais descritivo como `joao_silva.png`. Guarda esse nome pois ele será usado na configuração do autor.

Além disso, já prepare a imagem para ter obrigatoriamente o tamanho **150x150**. Esta não é uma restrição para o funcionamento do blog mas pode evitar dores de cabeça.

## Autor(configuração)

Agora o próximo passo é se adicionar ao grupo de authore nas configurações do blog. Para isso vá no arquivo `_config.yml` e se adicione como autor na seção de autores próximo da linha **10**. O campo authors neste arquivo de configuração é uma lista com todos os autores dos post identificados por uma chave única e com o conteúdo relativo a cada autor.

O conteúdo final deverá ficar algo do gênero

```yml
...
authors:
  arquivei:
  ...
  voce:
    name: João da Silva
    image: joao_silva.png
    about: Uma breve descrição sobre você que aparecerá na barra do lado esqueda. Seja sucinto, 200 caracteres é mais que o suficiente.
    social_networks:
      facebook: joao_silva
      github: joao_silva
      twitter: joao_silva
      linkedin: joao_silva
      email: joao.silva@arquivei.com.br
  ...
...  
```

Vale lembrar que na seção das redes sociais não existe campos obrigatórios, eles irão aparecer conforme você configurar. Deixe pelo menos o seu email pessoal caso alguém deseje entrar em contato para os posts que você fizer.

## Post(imagem)

A image do seu post irá aparecer tanto na publicação quando na página inicial. Ela não é obrigatória, mas sua publicação ficará muito mais elegante se você colocar uma. Além disso, não existe obrigatoriedade de tamanho mas algo em torno de **1280x850** será o padrão adotado para que os posts do blog.

Para linkar a imagem basta colocá-la no diretório `assets/img/posts`. É indiferente o nome que você der para o arquivo, mas tente colocar um nome mais descritivo como `meu_primeiro_post.png`. Guarda esse nome pois ele será usado na configuração do post

## Post(cabeçalho)

O cabeçaho possui algumas informações sobre o post como título, data de publicação, descrição, a imagem principal, tags e o autor. Todas as essas configurações são feitas manualmente e existe essa distinção do cabeçalho pois o que está fora do cabeçalho é o conteúdo dos posts.

A maioria dos itens é bem descritivo e intuítivo com exceção das tags que é uam lista de referências deste post em específico. Estas tags funcionam como uma definição de categoria dos posts servem para facilitar a busca de conteúdos relacionados à esta publicação.

```yml
---
layout: post
title: Meu Primeiro Post
date: 2017-11-30 13:32:20 +0300
description: Aqui você faz uma breve descrição do seu post que irá aparecer na home page do blog. Ten ser suficientemente sucinto. Algo em torno de 200 caracteres é suficiente.
img: meu_primeiro_post.png
tags: [Software, Design, QA]
author: seu_id
---
```

## Post(conteúdo)

O conteúdo do blog vem imediatamente depois do cabeçalho e deve ser escrito em markdown. Você provavelmente está acostumado a produzir conteúdos em markdown e caso tenha dúvidas basta consultar [o guia oficial do GitHub](https://guides.github.com/features/mastering-markdown/).

Eventualmente será necessário colocar imagens no seu post. Basta adicioná-las de forma similar à imagem principal do post com nomes intuitivos e no mesmo diretório e depois adicionar o trecho de código: `![Alt]({{site.baseurl}}/assets/img/posts/nome_do_arquivo.jpg)`.

## Post(publicar)

Para publicar o post basta fazer de forma análoga ao Citadel. É muito simples, basta dar commit e push que em segundos o seu post estará no ar, simples não!? O processo simplificado de publicação foi o principal motivo de escolhermos a tecnologia que está sendo usada.
