---
layout: post
title: Testando API Rest com Postman
date: 2018-03-29
categories: blog
img: postman-start.png
tags: [Postman, Testing API, QA, Test]
author: leonardo_camargo
brief: Como testar uma API Rest utilizando Postman + Newman
---
<style>
  article p {
    text-align: justify;
    text-indent: 25px;
  }

  article .center {
    margin-top: 30px;
    margin-bottom: 30px;
    text-align: center;
}

  .mb-1 {
    margin-bottom: 10px;
  }
</style>

Devido a evolução da área de desenvolvimento de software, aplicações desenvolvidas com base em serviços estão se tornando muito mais frequentes, deixando de dar o foco em aplicações com arquitetura monolitica onde tínhamos todas as funcionalidades do sistema unidas em único processo e passando a termos serviços a parte que se comunicam entre eles para chegar ao mesmo resultado com melhor desempenho, maior facilidade para manutenção e também a base de código menos extensa, facilitando a leitura do código.

Como toda aplicação, precisamos validar este serviço e garantir que o mesmo esteja atendendo os requisitos levantados.
Devido a uma demanda de um microserviço externo, surgiu a necessidade de realizarmos os testes na camada de serviço onde era necessario uma ferramenta que nos atendesse, com uma curva de aprendizado curta e que auxiliasse também no desenvolvimento.


### E é aí que entra o Postman!

![postman-go]({{site.baseurl}}/assets/img/posts/postman-go.gif)

# Postman

Postman é uma aplicação que auxilia no desenvolvimento de WebServices, principalmente REST API, realizando requisições em determinados endpoints, testes para validar os retornos das APIs, documentação, mocks de servidores e também relatórios após a realização dos testes.



<div class="center">
 <img src="/assets/img/posts/postman-info.png" class="img-fluid mb-1"/><br/>
</div>

### Vantagens Postman:
- Interface intuitiva o que facilita na configuração e execução dos testes
- Open Source
- Testes podem ser realizados em CLI (command-line interface ou Interface de linha de comandos) o que ajuda em ambientes que utilizam Integração Contínua
- Criação de servidores mockados para auxiliar na hora do desenvolvimento
- Transformar a requisição em algumas linguagens disponíveis

Legal, não? =D

# Taca lhe pau nesses testes

Antes existia uma extensão do Postman para o Google Chrome porém a extensão foi depreciada e eles recomendam baixar o app nativo que continua o suporte.
Para instalar, é só acessar o [Postman](https://www.getpostman.com/apps) e seguir os passos descritos dependendo do seu sistema operacional.

Para testes, utilizaremos a API do Postman que nos permite alterar e visualizar as nossas configurações da aplicação.
É necessário logar no Postman por esse [link](https://web.postman.co/) (caso não tenha login será necessário criar um usuário ou entrar com o email da google). Após isso basta acessar o seu workspace, clicar em Browse Integrations, View Details e Get API Key.

![postman-get-keys]({{site.baseurl}}/assets/img/posts/postman-get-key.png)

Dentro do app do Postman, iremos selecionar o método GET e informar a url "https://api.getpostman.com/me", No **Headers**, iremos informar a api-key que foi disponibilizada.
Então teremos no campo **"Key" = X-Api-Key** e no campo **"Value" a nossa api-key**.

![postman-header-get]({{site.baseurl}}/assets/img/posts/postman-headers+get.png)

Reparem que ao invés de colocar a nossa api-key gerada, colocamos ```postman-api-key```. Pelo fato de estarmos falando de uma senha, de algo sensível gerado exclusivamente pra você, é recomendado informarmos esses valores como variáveis de ambiente.
E como fazemos isso?

Simples! Basta clicar na engrenagem e clicar em "Manage Environments".

![postman-manage-environment]({{site.baseurl}}/assets/img/posts/postman-manage-environments.png)

Logo em seguida clicamos em Globals e informamos o nome do nosso value "Key", que é ```postman_api_key``` e o nosso "Value" que é a nossa **api-key** e salvamos =)
Após isso, é só clicarmos em **Send** e teremos a resposta da nossa API com o nosso ID.
```{
    "user": {
        "id": 4011658
    }
```
Com isso conseguimos fazer nosso primeiro teste. Basta irmos na aba "Tests". O Postman nos dá alguns testes default para conseguirmos validarmos o nosso retorno. Vamos validar se o nosso retorno é uma resposta 200. Basta clicarmos no Snippet **Status Code: Code is 200** e clicarmos em **Send** novamente. Se clicarmos em "Tests Results" iremos ver o resultado do nosso teste que passou com sucesso =)

![postman-result-200]({{site.baseurl}}/assets/img/posts/postman-200.png)

Outro teste que podemos fazer é validar se o nosso id de usuário em uma outra requisição vai continuar sendo o mesmo. Basta adicionarmos essas linhas de códigos:

``` js
pm.test("Compare user id", function () {
    var jsonData = pm.response.json();
/*convertendo a nossa resposta para json */
    pm.expect(jsonData.user.id).to.be.equal(4011658);
/* comparando o retorno do id que está dentro de user com  id que foi retornado no primeiro teste */
});
```
Os testes são realizados com Chai, uma biblioteca que realiza algumas asserções.

# Newman

O Newman nada mais é que uma versão minificada do Postman para ser rodado em linha de comando. Isso nos ajuda a utilizar nossos testes em ambientes de CI (Integração Contínua). Para utilizarmos teremos que instalá-lo em nossa máquina. Ele foi escrito e roda em NodeJS isso facilita pois pode ser utilizado em qualquer sistema. Para instalar o NodeJS basta acessarmos a página do [node](https://nodejs.org/en/). Após instalar o NodeJS basta rodarmos o comando:

 ``` npm install newman --global; ```

Para rodarmos nossos testes em linha de comando, precisamos exportar nossos testes e também a nossa variável global. Para exportar os nossos testes só precisamos salvar nossos testes, precisamos ir no Postman e clicar em **Save**, informando um nome para nossa Collection que seria todos os nossos testes. Após isso basta clicar em Export e salvar na pasta desejada.

Para exportar a nossa variável de ambiente basta irmos em Manage Environments e clicar em "Download as JSON".
Para rodar nosss testes, precisamos rodar o seguinte comando:

```newman run nome_do_arquivo_dos_tests.json -G nome_arquivo_variaveis.json```

Após rodar o nosso teste, ele nos dá uma tabela fornecendo o resultado dos testes:

![postman-result-200]({{site.baseurl}}/assets/img/posts/postman-resultado-teste.png)

Podemos também, exportar nossos testes para html, cli, json ou junit. basta adicionarmos ```-r html``` e será criado uma pasta "newman" no local da collection com o relatorio gerado.

# Conclusão

Postman é uma ferramenta de fácil utilização e quando bem utilizada consegue ajudar não só nos testes mas também no desenvolvimento da aplicação. Muitas pessoas conhecem o Postman apenas para validar a chamada, ver a resposta mas conseguimos ver que podemos ir além, podemos testar essas chamadas fazendo várias asserções em javascript e gerar métricas, e além disso, conseguimos em um ambiente de CI termos um teste para contribuir no build.
Uma ferramenta completa que consegue contribuir para todas as áreas de desenvolvimento: QA, Frontend, Backend e Devops.
