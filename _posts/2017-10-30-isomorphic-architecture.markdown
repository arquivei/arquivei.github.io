---
layout: post
title: Arquitetura Isomórfica
date: 2017-11-30
categories: blog
img: isomorphic-architecture.png
tags: [Javascript, Isomorphic, Architecture]
author: olinda
brief: O meu principal objetivo com a arquitetura isomórfica é promover ao cliente uma renderização instantânea vinda do servidor e, ao mesmo tempo, proporcionar uma boa experiência de uso com a interatividade do javascript. Afinal, quando se trata de loading o mundo tem pressa!
---

Quando me refiro a **Arquitetura Isomórfica** esta faz referência aos conceitos de **Javascript Isomórfico** ou **Javascript Universal** propostos pelo Airbnb em novembro de 2013. No universo da Arquivei quando dizemos Arquitetura Isomórfica isso traduz que o frontend é renderizado tanto no microserviço como no browser do cliente via javascript. Toda requisição que acontece no nosso frontend é renderizada duas vezes, no microserviço e no cliente.

# Contexto

Há cerca de 1 ano atrás, em meados de 2016, me surgiu a proposta de arquitetar um projeto do desmembramento de um monolito em um arquitetura de frontend-backend. Naquela época o AngularJS já tinha iniciado a sua decadência, o React era modinha, ES6 era o termo mais legal de se falar e a maioria das aplicações web eram uma Single Page Application(SPA).

# Introdução

A minha ideia inicial era criar um frontend que fosse demasiadamente simples de usar, que a criação e acompanhamento das features no projeto fosse de maneira ágil, que proporcionasse uma excelente experiência para o usuário e que garantisse o pleno funcionamento para o usuário(sem erros de runtime). Logicamente, alguns desses objetivos ainda não foram alcançados mas eu gostaria de dar um overview do ponta pé inicial do frontend na Arquivei.

No começo do projeto eu estudei diversas arquiteturas de frontend e a maioria dos modelos apresentados tinham pelo menos 10 anos de idade. Posso afirmar ainda que em 2006 o cenário web era muito diferente do moderno. A Vila Bol ainda era um sucesso, o Orkut era o site do momento, o MSN era o messenger da década, o bate-papo da Uol era o point da paquera e o mais próximo de um smartphone era o Nokia 2280.

Sobre arquitetura de frontend, o modelo moderno mais encontrado eram as SPAs e correspondiam ao cenário onde o usuário fazia uma requisição, um `index.html` era retornado com vários assets e um body praticamente em branco e toda a aplicação seria carregada à partir dali.

![isomorphic-architecture_01]({{ site.baseUrl }}/assets/img/posts/isomorphic-architecture_04.png){:class="img-fluid"}

Depois de um intenso estudos sobre o assunto parecia que o único caminho trilhável era o de criar um aplicação SPA em React. Esse cenário mudou um tanto depois de bater um papo com o pessoal da [Elo7](http://engenharia.elo7.com.br/) e ver que eles usavam uma arquitetura um tanto quanto diferente. Aparentemente eles usavam o frontend como um serviço de renderização de HTML, dividiam sua aplicação em componentes e ainda rodavam esses mesmos componentes no browser do cliente. Inicialmente isso soou muito mágico mas de certa forma abriu a minha mente para propor novos modelos de arquitetura para o desmembramento do monolito.

# Primeira POC

Meu primeiro limitante para progredir com uma primeira POC foi o de entender como é possível implementar uma aplicação que renderiza HTML tanto no servidor como via assets. Depois de algumas pesquisas no Google o artigo que me deu um rumo para o caminho que eu procurava foi um artigo do Airbnb sobre [Javascript Isomórfico](https://medium.com/airbnb-engineering/isomorphic-javascript-the-future-of-web-apps-10882b7a2ebc). O artigo publicado pelo Airbnb é bem claro e objetivo. Ele dá um ideial bem conceitual sobre o que é javascript isomórfico e fazia muito sentido para criar o modelo de arquitetura que eu procurava.

Nesse momento eu tirei a primeira conclusão sobre o conceito que eu buscava para criar a arquitetura de frontend, uma arquitetura isomórfica.

> Arquitetura Isomórfica é uma estratégia que proporciona rodar uma mesma aplicação singular tanto no cliente quanto no servidor.

## Ideias

Uma vez tendo postulado a minha ideia sobre o projeot, comecei a escrever a minha primeira POC. Nesse momento eu queria usar uma ferramenta que me proporcionasse a implementação desta arquitetura sem muita dificuldade e com pouco código. Em resumo, eu procurava um framework que fizesse parte do trabalho para mim.

Na época o universo de ferramentas que proporcionavam a implementação estratégia era bem escasso. Vamos dar um overview sobre as principais:

- O Angular estava em decadência e a sua versão 1 não proporcionava isomorfismo;
- O VueJS estava na sua primeira versão e a estratégia isomórfica era prometida para a segunda versão;
- O React já proporcionava o isomorfismo e era uma tecnologia bem **Hipe** para o momento;
- O polymer... bom, até hoje não sei quase nada sobe ele.

A minha ideia inicial era criar um microserviço de frontend de tal forma que ele ficasse devidamente isolado, fosse testado de ponta a ponta, fizesse com que a equipe de backend jamais precisasse digitar qualquer trecho de código javascript, css ou html.

## Server Side

O engraçado é que o primeiro desenho da arquitetura de frontend como um microserviço que eu fiz foi demasiadamente simples e é o mesmo esquema até hoje na Arquivei. **A beleza da simplicidade.**

O microserviço consiste basicamente em uma aplicação NodeJS que recebe um devido estado e devolve uma string com todo o HTML que, por sua vez, é entregue ao cliente final sem nenhuma alteração.

![isomorphic-architecture_01]({{ site.baseUrl }}/assets/img/posts/isomorphic-architecture_05.png){:class="img-fluid"}

A ideia de renderizar todo o HTML do `DOCTYPE` até `</body>` era simplesmente deixar tudo relativo ao frontend fora da equipe que desenvolvia o monolítico/backend. Até mesmo o gerenciamento de assets e o deploy ficaria por conta de equipe de frontend.

## Client Side

Uma vez especificado como funcionaria o microserviço bastava gerar os assets dos códigos isomórficos escritos. Em resumo eu dividi a aplicação em rotas e cada rota tem o seu javascript específico. O javascript que está presente em mais de uma rota é inserido em um `common.js`.

Na minha primeira POC foi mais difícil configurar um bundler parar gerar os assets do que separar logicamente a aplicação para rodar de maneira isomófica. O bundler/builder/magic que usei foi o [Webpack](https://webpack.js.org/) e eu realmente não tinha um bom entendimento de como as coisas funcionavam. Como o propósito era fazer uma POC eu apenas confiei na mágica.

Hoje eu tenho um entendimento muito bom sobre o **Webpack** e entendo que este é praticamente o estado da arte da computação web moderna. Aqueles que não sabe muito sobre recomendo verem os diversos boilerplates que existem no GitHub.

## Resultados

Bom, o primeiro resultado que eu obtive foi a de uma página que carregava mesmo sem o javascript. Para mim isso foi muito gratificante uma vez que a página carregava todo o DOM instantâneamente. O javascript demorava cerca de 1 ssegundo para carregar e depois disso a aplicação estava pronto para ser usada. O ponto é que esse atraso do javascript era invisível para o usuário e parecia que a página era instantâneamente carregada e não tinha nenhum carregamento depois disso.

O usuário simplesmente abria a página e todo o conteúdo já estava lá, instantâneamente. Como o tempo de reação do usuário era maior que 1 segundo não tinha problema demorar um pouco mais para carregar o javascript.

Um ano mais tarde eu reproduzi essa mesma POC para um minicurso sobre microserviços que foi ministrado pela Arquivei na [Semana da Computação](https://semcomp.icmc.usp.br/20/) da USP de São Carlos e ele pode ser encontrado no meu github em um projeto chamado
[arquivei-semcomp-frontend](https://github.com/toruticas/arquivei-semcomp-frontend).

## Performance

Esta seção vai falar um pouco das premissas que fiz quando criei a primeira POC com o resultado final da aplicação em produção. A minha ideia é mostrar um pouco do que previamos e o seu resultado final.

A primeira vez que discuti performance foi depois de ter implementado a primeira POC pois agora eu teria uma aplicação reduzida para conseguir dissertar sobre várias coisas. Na minha primeira POC foi possível validar que a visualização realmente era carregada intantâneamente, ou seja, no primeiro render da página. Após isso o javascript demoraria cerca de 1 segundo para carregar para a aplicação começar a funcionar.

Com base no que vimos, começamos a criar hipóteses sobre a nossa principal tela do sistema, a NFeList, que é a tela principal da Arquivei. Nesta tela é possível encontrar informações de algumas notas fiscais eletrônicas, os diversos vários fluxos relacionados à parte contábil/jurídica da nota e algumas personalizações do usuário. No monolito essa tela é contruida por um view helper no framework que usamos e depois é modificada diversas vezes com o Jquery. A grosso modo, cada feature que entrava nessa tela um código jquery era adicionado dentro do espaguete HTML.

Essa é era a visão da NFeList no monolito. Em resumo, cada ponto vermelho é um [reflow](https://developers.google.com/speed/articles/reflow) forçado no DOM e é um gargalo na perfomance. A área em azul representa o tempo de carregamento, a área em amarelo é a execução dos scripts, a área em roxo é o rendering e a área em verde é o painting.

![isomorphic-architecture_01]({{ site.baseUrl }}/assets/img/posts/isomorphic-architecture_01.png){:class="img-fluid"}

O segundo gráfico apresentado é o da NFeList no modelo isomórfico, já em produção feito em React/Redux. No gráfico já conseguimos concluir que a aplicação é renderizada em cerca de 1.8 segundos e o reflow forçado pelo React acontece entre 3 à 3.5 segundos. Peço desculpas pelo longo tempo de respostas pois nesse gráfico o javascript carregado ainda não estava gzipado.

![isomorphic-architecture_02]({{ site.baseUrl }}/assets/img/posts/isomorphic-architecture_02.png){:class="img-fluid"}

No lado do microserviço temos o tempo de resposta é de cerca de 70 milisegundos no ápice do seu uso que é às 16h na primeira semana do mês. Estes 70 milisegundos é o tempo que o Load Balancing leva para enviar a requisição e receber a resposta. Além disso, o uso da máquina é bem abaixo do que esperávamos. Obs: o pico é um deploy!

![isomorphic-architecture_02]({{ site.baseUrl }}/assets/img/posts/isomorphic-architecture_03.png){:class="img-fluid"}

# Conclusão

No que tange à performance ainda há um extenso estudo para ser feito pois carregamento inicial é um grande gargalo devido ao React e alguns plugins que deixaram a aplicação mais pesada do esperávamos. Uma possível melhoria talvez seja remover algumas dependências de propósito geral que deixaram a aplicação pesada.

Além disso, a instância da máquina que roda o microserviço de frontend possui uma rede um pouco mais limitada que acaba deixando o tempo de resposta um pouco elevado. A nossa ideia é colocar o microserviço em um Kubernetes que vai usar uma instância que possua uma rede mais elástica do que a usada pelo microserviço de frontend.

Quanto à **Arquitetura Isomórfica** não tenho sombra de dúvidas que foi a melhor escolha que fiz, uma vez que desenvolvemos praticamente o mesmo código para o microserviço e para o assets finais, além de melhorar a performance, proporcionar um workflow bem definido e promover uma usabilidade muito mais agradável ao usuário final.

Acredito fortemente que, mesmo com os browsers modernos, a estratégia isomórfica garante uma boa experiência para o usuário e que talvez seja o destino de todos WebApps.
