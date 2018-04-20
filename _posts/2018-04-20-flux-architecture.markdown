---
layout: post
title: Arquitetura Flux
date: 2018-04-20
img: flux-architecture.png
tags: [Javascript, Flux, Architecture, MVC, Redux]
author: olinda
brief: A arquitetura flux é um modelo unidirecional de fluxo de dados e é mais um padrão do que um framework. Acredito que esta arquitetura revolucionou a maneira de pensar o frontend.
---

A arquitetura flux foi um conceito apresentado pelo facebook e você pode ver tudo mais detalhado no site oficial [https://facebook.github.io/flux/](https://facebook.github.io/flux/). A arquitetura flux é um modelo unidirecional de fluxo de dados e é mais um padrão do que um framework.

# O que é flux?

Flux é uma arquitetura de fluxo de dados desenhado pelo Facebook em conjunto com o react que propõe deixar o os caminhos dos dados e as suas atualizações mais explícita e compreensível. Isso permite que as aplicações sejam mais simples de desenvolver, rastrear bugs e arrumá-los.

# Contexto histórico

Acredito que o que promoveu a criação de um novo modelo de fluxo de dados foi o excessivo uso da arquitetura MVC para o cliente, sobretudo usado pelo Angular v1 (não posso afirmar as demais versões porque eu não conheço). O MVC resolve o problema de aplicações web de propósito geral e estava sendo usado largamente em aplicações SPA, deixando as páginas mais lentas, difíceis de desenvolver e na existência de bugs a vida do programador se transformava num inferno.

# Visão geral do MVC

O MVC é padrão de arquitetura de software usualmente conhecido na web que separa logicamente os dados e suas regras de negócio e da apresentação (modelo de três camadas). Acredito que como em qualquer padrão de arquitetura de software o principal propósito do MVC é a reusabilidade de código e uma separação lógica das suas necessidades.

Vou tentar dar um visão bem superficial de como funciona o modelo MVC mas acredito que não é nada muito mais complexo do que vou falar. Vou ser um tanto quanto sucinto pois o foco deste artigo não é o MVC. Você consegue achar um aprofundamento muito melhor em diversos artigos pela internet.

A **View** é a camada responsável por representar os dados de alguma forma podendo ter várias visões sobre um mesmo dado. O **Model** é a representação dos dados da aplicação bem como as suas regras de negócio e responsabilidades. O **Controller** faz a multiplexação da entrada fazendo um meio de campo entre o **Model** e a **View**.

A visão geral de um MVC é algo como descrito na imagem abaixo.

![flux-architecture-mvc-simple]({{ site.baseUrl }}/assets/img/posts/flux-architecture/mvc-simple.svg){:class="img-fluid"}

Mas quando estou falando de fluxo de dados o MVC consegue ser muito mais complicado do que parece e forma um emaranhado de caminhos que o código percorre dependendo da visualização e multiplexação que acontecem no controller.

Este padrão de arquitetura de software é o mais encontrado em frameworks web e você não conseguirá escapar deles para os principais da internet como o Rails, Django, Laravel e Spring.

# Flux

Para descrever a melhor a arquitetura flux eu vou compará-la ao modelo MVC tradicional. Vou reforçar ainda que este será um exemplo de um modelo MVC client-side com as experiências que tive com Angular v1.

## MVC client-side

Em uma aplicação MVC o usuário, durante a interação, provoca um evento processado em um controller que efetua alguma regra de negócio sobre um model. Este, por sua vez,  atualiza a view e esta por fim pode buscar modelos nessa visualização.

## Escalando o MVC client-side

O problema é que quanto mais a aplicação crescer, menos escalável estará essa própria aplicação pois a dependência entre controllers, models e views se tornará tão complexa que tornará difícil a criação de novas funcionalidades ou até mesmo a manutenção das existentes.

Com um exemplo simples de dois controllers, dois models e duas views a dependência entre as camadas já fica realmente complexa para entender. Talvez no código isso pareça ser simples mas não é tão escalável quanto você imagina.

Expandindo esse problema para um bug fica realmente difícil de compreender qual o exato caminho pelo qual uma ação executada passou e se tratando de interação com usuário esse grafo de dependências fica mais complexo ainda.

![flux-architecture-mvc-simple]({{ site.baseUrl }}/assets/img/posts/flux-architecture/mvc-complex.svg){:class="img-fluid"}

## Introdução ao Flux

O Flux traz um conceito de fluxo unidirecional de dados onde a interação do usuário vai provocar um evento no formato de uma action específica. Essa ação vai despachar o evento para a modificação de um dados, que vai modificar o store de dados e, por fim, este vai enviar um evento de atualização da visualização. Ao final a visualização vai consumir os dados das mais variadas formas.

![flux-architecture-mvc-simple]({{ site.baseUrl }}/assets/img/posts/flux-architecture/flux-simple.svg){:class="img-fluid"}

## Escalando a arquitetura Flux

Expandindo o conceito sobre o store ele possui toda a lógica de neǵocio da aplicação e pode ser composto por vários stores onde cada store é responsável por um domínio específico da aplicação. Além disso, a visualização já consegue ser composta por vários níveis de visualização que irão para o cliente e, talvez, em somente uma parcela exista um ponto passível de chamar uma ação.

Escalando essa aplicação conseguimos ter uma visão singular de como funcionam os possíveis caminhos que os dados percorrem para formar uma visualização para o usuário final. Acho que vale a pena destacar como fica claro a reusabilidade de código e a separação lógica tanto dos domínios quanto dos componentes visuais.

![flux-architecture-mvc-simple]({{ site.baseUrl }}/assets/img/posts/flux-architecture/flux-complex.svg){:class="img-fluid"}

# Conclusões

A arquitetura flux tem uma série de propriedades que a torna singular e promove uma série de vantagens quando comparada às demais arquiteturas de software. Essa arquitetura mantém todos os dados centralizados em um único ponto e faz com que o fluxo de dados seja explícito, fácil de desenvolver e dar manutenção (sobretudo pela facilidade de localizar bugs quando eles acontecem).

Além disso, existe uma série de valores mais explícitos sobre a arquitetura flux. Vou citar alguns deles aqui.

## Sincronia

Toda as modificações de dados são dadas de forma síncrona e quando acontece alguma operação assíncrona, como por exemplo ajax, estas disparam uma action que executa todo o fluxo previsto pela arquitetura. Além disso, é possível que suas ações ainda disparem ações assíncronas da API (do browser ou do node). Em resumo, quando há uma chamada tanto síncrona quanto assíncrona o fluxo dos dados fica bem explícito dentro da aplicação e fácil de ser depurado quando há lógica incorretas no seu código.

## Inversão de controle

Nenhuma outra parte do código precisa conhecer como modificar o estado da aplicação pois os stores fazer isso internamente através das ações, ou seja, toda a lógica de atualizar os dados está contida no store. Além disso, como as atualizações dos stores só acontecem em resposta às ações, isso sincronamente, testar stores se torna demasiadamente simples pois você terá um "estado inicial", uma ação e um "estado final".

## Ações mais semânticas

Como os stores atualizam a si mesmos em resposta às ações, as ações tendem a ser semanticamente mais descritivas. Por exemplo, em uma aplicação flux de notas fiscais eletrônicas você poderá abrir a visualização de uma nota através de uma ação "TOGGLE_VIEW_NFE", passando o identificador da nota fiscal como argumento. A ação em si não sabe como efetuar a atualização mas descreve plenamente o que espera que seja feito.

Por causa desta propriedade você raramente terá que trocar as suas ações, apenas como o store responde a essas ações. Quanto mais a sua aplicação se aproxima de um conceito de um Saas, quanto mais pontos de interação essa aplicação tenha tenha, a ação "TOGGLE_VIEW_NFE" se tornará semanticamente válida.


# Implementações

Existem uma série de aplicações que implementam a arquitetura flux tais como Flux, Elm-lang, Redux, MobX, etc. Na Arquivei nós usamos Redux pela sua característica semântica e imperativa. Com isso, criamos uma arquitetura bem específica para separar os dados da aplicação por completo. Mais recentemente nós estamos usando [redux-rematch](https://github.com/rematch/rematch) para diminuir a implementação de código desnecessário.

Em um próximo papo falaremos sobre a nossa arquitetura, que chamamos de Connect-Partials, e de como o redux-rematch está facilitando a nossa vida. Por agora, agradeço a leitura! :)
