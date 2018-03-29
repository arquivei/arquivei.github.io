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


  article .center {
    margin-top: 30px;
    margin-bottom: 30px;
    text-align: center;
}

</style>

Com a evolução da área de desenvolvimento de software, aplicações desenvolvidas com base em serviços estão se tornando muito mais frequentes. O foco em aplicações com arquitetura monolitica vem diminuindo e, onde antes tínhamos todas as funcionalidades do sistema unidas em um único processo, passamos a ter serviços especializados que se comunicam entre si, afim de chegar ao mesmo resultado trazendo melhor desempenho, maior facilidade para manutenção e bases de código menos extensa.

Como toda aplicação, precisamos validar estes serviços e garantir que os mesmos estejam atendendo os requisitos levantados.
Devido a demanda de um microserviço externo, surgiu a necessidade de realizarmos, na Arquivei, os testes na camada de serviço. Para isso, era necessario uma ferramenta com uma curva de aprendizado curta para a equipe de qualidade e que auxiliasse os desenvolvedores em seu processo criativo.


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
É necessário logar no Postman por esse [link](https://web.postman.co/) (caso não tenha login será necessário criar um usuário ou entrar com o email da google). Após isso acesse o seu workspace, clicar em Browse Integrations, View Details e Get API Key.

![postman-get-keys]({{site.baseurl}}/assets/img/posts/postman-get-key.png)

Dentro do app do Postman, iremos selecionar o método GET e informar a url "https://api.getpostman.com/me", No **Headers**, iremos informar a api-key que foi disponibilizada.
Então teremos no campo **"Key" = X-Api-Key** e no campo **"Value" a nossa api-key**.

![postman-header-get]({{site.baseurl}}/assets/img/posts/postman-headers+get.png)

Reparem que ao invés de colocar a nossa api-key gerada, colocamos ```postman-api-key```. Pelo fato de estarmos falando de uma senha, de algo sensível gerado exclusivamente pra você, é recomendado informarmos esses valores como variáveis de ambiente.
E como fazemos isso?

Simples! É só clicar na engrenagem e clicar em "Manage Environments".

![postman-manage-environment]({{site.baseurl}}/assets/img/posts/postman-manage-environments.png)

Logo em seguida clicamos em Globals e informamos o nome do nosso value "Key", que é ```postman_api_key``` e o nosso "Value" que é a nossa **api-key** e salvamos =)
Após isso, é só clicarmos em **Send** e teremos a resposta da nossa API com o nosso ID.
```{
    "user": {
        "id": 4011658
    }
```
Com isso conseguimos fazer nosso primeiro teste. Clique na aba Tests". O Postman nos dá alguns testes default para conseguirmos validar o nosso retorno. Vamos validar se o nosso retorno é uma resposta 200. Basta clicar no Snippet **Status Code: Code is 200** e clicar em **Send**. Para validar os resultados do teste, podemos ir até "Tests Results" e validar se a resposta da assertion foi atendida.

![postman-result-200]({{site.baseurl}}/assets/img/posts/postman-200.png)

Outro teste que podemos fazer é validar se o nosso id de usuário em uma outra requisição continuará sendo o mesmo. Fazemos isso adicionando o seguinte código:

``` js
pm.test("Compare user id", function () {
    var jsonData = pm.response.json();
/*convertendo a nossa resposta para json */
    pm.expect(jsonData.user.id).to.be.equal(4011658);
/* comparando o retorno do id que está dentro de user com  id que foi retornado no primeiro teste */
});
```
Os testes são realizados com o framework [Chai](http://www.chaijs.com/) que é uma biblioteca feita para node. O Postman utiliza o NodeJS para criar um backend e executar os testes criados. O Chai disponibiliza diversas assertions bem intuitivas para validar as respostas.

# Newman

O Newman nada mais é que uma versão minificada do Postman para ser rodado em linha de comando. Isso nos ajuda a utilizar nossos testes em ambientes de CI (Integração Contínua). Para utilizarmos teremos que instalá-lo em nossa máquina. Ele foi escrito e roda em NodeJS isso facilita pois pode ser utilizado em qualquer sistema. Para instalar o Newman é necessário ter o gerenciador de pacotes **npm**. Com o NodeJS instalado, é só rodar o comando:

 ``` npm install newman --global; ```

Para rodarmos nossos testes em linha de comando, precisamos exportar nossos testes e também a nossa variável global. Para exportar os nossos testes só precisamos salvar nossos testes, precisamos ir no Postman e clicar em **Save**, informando um nome para nossa Collection que seria todos os nossos testes. Após isso é só clicarmos em Export e salvar na pasta desejada.

Para exportar a nossa variável de ambiente acesse o Manage Environments e clique em "Download as JSON".
Para rodar nossos testes, precisamos rodar o seguinte comando:

```newman run nome_do_arquivo_dos_tests.json -G nome_arquivo_variaveis.json```

Após rodar o nosso teste, ele nos dá uma tabela fornecendo o resultado dos testes:

![postman-result-200]({{site.baseurl}}/assets/img/posts/postman-resultado-teste.png)

Podemos também, exportar nossos testes para html, cli, json ou junit adicionando ```-r html``` no final do comando. Será criado uma pasta "newman" no local da collection com o relatorio gerado.

# Conclusão

Postman é uma ferramenta de fácil utilização e quando bem utilizada consegue ajudar não só nos testes mas também no desenvolvimento da aplicação.Muitas pessoas conhecem o Postman apenas para validar a chamada, ver a resposta. Mas conseguimos ir além: podemos testar nossas chamadas através de várias asserções em javascript e gerar métricas. Além disso, conseguimos em um ambiente de CI termos um teste para contribuir no build.
Uma ferramenta completa que consegue contribuir para todas as áreas de desenvolvimento: QA, Frontend, Backend e Devops.
