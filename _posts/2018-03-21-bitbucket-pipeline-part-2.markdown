---
layout: post
title: A jornada pela integração contínua - Parte 2
date: 2018-03-21
categories: blog
img: bitbucket-pipeline.png
tags: [Automation, Bitbucket, Pipeline, Docker, gcloud, aws]
author: flipper
brief: Como atingir um processo de integração contínua utilizando a ferramenta de Pipelines do Bitbucket.
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

  article a {
    text-decoration: none;
  }

  pre {
    background-color: #2d2d2d;
    border-radius: 4px;
  }

  code {  
    font-size: 95%;
    line-height: 140%;
    white-space: pre;
    white-space: pre-wrap;
    white-space: -moz-pre-wrap;
    white-space: -o-pre-wrap;
  }

  span.code {
    border-radius: 4px;
    color: white;
    font-family: Monaco, monospace;
    font-size: 95%;
    line-height: 140%;
    white-space: pre;
    white-space: pre-wrap;
    white-space: -moz-pre-wrap;
    white-space: -o-pre-wrap;
    background-color: #2d2d2d;
    padding: 2px 10px;
  }

  .mb-1 {
    margin-bottom: 10px;
  }
</style>

<center><i>O build quebrou, como faz?</i></center>

<a href="{{site.baseurl}}/bitbucket-pipeline-part-1" target="_blank">No post anterior</a> expliquei um pouco do processo criativo e a primeira solução por trás da jornada pela integração contínua. Dessa vez vou falar um pouco sobre a solução final que atendeu todas as expectativas, o Bitbucket Pipeline.

O <a href="https://bitbucket.org/" target="_blank">Bitbucket</a> é uma ferramenta da <a href="https://www.atlassian.com/" target="_blank">Atlassian</a> semelhante ao <a href="https://github.com/" target="_blank">Github</a>, que cuida do versionamento de código da aplicação num repositório compartilhado, também conhecido como SCM. Um dos adicionais do Bitbucket é o <a href="https://bitbucket.org/product/features/pipelines" target="_blank">Pipeline</a>, uma ferramenta de integração contínua disponível por padrão em todos os planos.

<small>Ferramentas similares existem em produtos do <a href="https://github.com/marketplace/category/continuous-integration" target="_blank">Github</a> e <a href="https://about.gitlab.com/features/gitlab-ci-cd/" target="_blank">Gitlab</a> também.
</small>

# O Pipeline

A ideia do Pipeline é extremamente simples: ele executa um conjunto de instruções pré-definidas em um contêiner do <a href="https://www.docker.com/" target="_blank">Docker</a> sobre um conjunto pré-definido de branches. Como assim?


Para configurar o Pipeline basta adicionar um arquivo chamado <span class="code">bitbucket-pipelines.yml</span> na raiz do seu projeto. Uma configuração básica exige:

* Uma imagem base do Docker (Se não providenciar uma, será usada a padrão e, não queremos isso)
* Um trigger
* Um script

<div class="center">
 <img src="/assets/img/posts/bitbucket-pipeline_01.png" alt="Um exemplo simples do bitbucket-pipelines.yml" class="img-fluid mb-1"/><br/>
 <small>Um exemplo simples do bitbucket-pipelines.yml</small>
</div>

Na exemplo acima temos um trigger default, que será executado em todas as branches do projeto, com um script que realiza a rotina de testes. Isso automatiza o processo de testes e garante que nenhum código seja enviado para a produção com testes unitários quebrados que passaram despercebidos. Esse script é extremamente similar a um script que rodamos no nosso repositório, o que garante a cobertura de testes do projeto.

<div class="center">
 <img src="/assets/img/posts/bitbucket-pipeline_02.png" alt="Exemplo da tela do Pipeline" class="img-fluid mb-1"/><br/>
 <small>Exemplo da tela do Pipeline</small>
</div>

A principal vantagem do Pipeline é a execução remota do script com base em triggers pré-definidos. Como a execução se baseia num contêiner do Docker, as possibilidades são virtualmente ilimitadas. Além de todas as óbvias vantagens do serviço, optei por usar o Pipeline pois ele já era utilizado pela equipe Frontend para executar a rotina de testes do projeto. Tudo o que eu precisava fazer era adicionar a integração contínua e gerar as imagens.

# A solução final

O objetivo era automatizar o processo de forma genérica, deixando o mínimo de trabalho possível quando o resto da Engenharia migrasse para o serviço (o Frontend era a única equipe que fazia uso do Pipeline). Como o Pipeline roda uma instância do Docker, bastava criar uma imagem que tivesse todas as dependências necessárias incluídas, assim times diferentes poderiam apenas importar a imagem nos seus respectivos pipelines e subir as imagens automaticamente com o mínimo de configuração. Uma vez criada essa imagem, o próximo passo era fazer o build dentro da instância do Pipeline e posteriormente fazer o push para os repositórios na cloud.

Ao final, o Dockerfile criado para servir de imagem base no Pipeline para toda a Engenharia ficou assim:

```yml
# Install python, needed for both gcloud and awscli
FROM debian:stretch as arqPy

ENV PYTHON_VERSION 2.7

# Update and install minimal.
RUN apt-get update \
       --quiet \
   && apt-get install \
       --yes \
       --no-install-recommends \
       --no-install-suggests \
   build-essential \
   git \
   python$PYTHON_VERSION \
   python$PYTHON_VERSION-dev \
   python-pip \
   python-setuptools \
   curl \
   gnupg \
   tar

# Update pip and set up virtualenv.
RUN pip install \
   -U pip

# Install gcloud and awscli for repository management
FROM arqPy as arqTools

ENV GCLOUD_SDK_URL=<gcloud_sdk_download_url>

RUN \
 curl -o /opt/google-cloud-sdk.tar.gz ${GCLOUD_SDK_URL} \
 && tar -xvf /opt/google-cloud-sdk.tar.gz -C /opt/ \
 && /opt/google-cloud-sdk/install.sh -q

RUN pip install awscli
RUN export PATH=~/bin:$PATH

# Default command.
CMD ["bash"]
```
<small>Substitua a url do GOOGLE\_CLOUD\_SDK pela url de download do gcloud da última versão.</small>

Com essa imagem, que está disponível publicamente no <a href="https://hub.docker.com/r/arquivei/pipeline/" target="_blank">Docker Hub da Arquivei</a>, toda a Engenharia poderia fazer o build e publicar nos seus respectivos repositórios de imagens. Além disso já existe a possibilidade de uma integração automática tanto com triggers (por exemplo: branches e ativação manual), como com crons nas próprias configurações do Pipeline no repositório.

O script final do build do Frontend dentro do <span class="code">bitbucket-pipeline.yml</span> ficou assim:

```yml
image: node

options:
 docker: true

pipelines:
 custom:
   prod:
     - step:
        image: arquivei/pipeline
        name: Docker Build - Production
        script:
          # # # # # # # # # # # # # # # # # # # # # # # # # # # # # #
          - echo $ENV_PROD | base64 --decode --ignore-garbage > ./app/.env
          - docker build -t gcr.io/${GCLOUD_PROD_PROJECT}/frontend/app:${BITBUCKET_BUILD_NUMBER}-$(git rev-list HEAD --max-count=1 --abbrev-commit) -t ${AWS_PROD_REGISTRY_BASE_URL}/frontend/app:${BITBUCKET_BUILD_NUMBER}-$(git rev-list HEAD --max-count=1 --abbrev-commit) -f ./app/Dockerfile .
          # # # # # # # # # # # # # # # # # # # # # # # # # # # # # #
          - source /opt/google-cloud-sdk/path.bash.inc
          - echo $GCLOUD_PROD_KEYFILE | base64 --decode --ignore-garbage > ./gcloud-api-key.json
          - gcloud auth activate-service-account --key-file ./gcloud-api-key.json
          - gcloud config set project $GCLOUD_PROD_PROJECT
          - gcloud docker -- push gcr.io/${GCLOUD_PROD_PROJECT}/frontend/app
          # # # # # # # # # # # # # # # # # # # # # # # # # # # # # #
          - mkdir ~/.aws
          - echo $AWS_CREDENTIALS | base64 --decode --ignore-garbage > ~/.aws/credentials
          - echo $AWS_CONFIGS | base64 --decode --ignore-garbage > ~/.aws/config
          - $(aws ecr get-login --region ${AWS_DEFAULT_REGION} --no-include-email --profile prod)
          - docker push ${AWS_PROD_REGISTRY_BASE_URL}/frontend/app
```

Com esse script, que pode ser compartilhado entre toda a Engenharia com pequenas ou nenhuma mudança, todo o processo de publicação das imagens é feito de forma automática. As variáveis de ambiente <span class="code">${VAR}</span> são configuradas na própria interface do Pipeline e podem ser configuradas como globais (para todos os repositórios do Arquivei), ou como locais (apenas para o repositório específico desejado). Esse script já gera a imagem de produção, pronta para o deploy, mas a imagem de QA/teste é gerada simplesmente trocando a variável <span class="code">${ENV\_PROD}</span> por <span class="code">${ENV\_DEV}</span>, graças às configurações das variáveis de ambiente.

# Conclusão

Tentar diferentes abordagens para solucionar esse problema trouxe uma boa bagagem e devo dizer que aprendi a usar o Docker com tranquilidade apenas depois dessa jornada. O Pipeline é uma ferramenta simples, mas sua integração com o Docker a torna, na minha opinião, extremamente mais poderosa do que aparenta ser. Sua configuração é absurdamente simples, suas possibilidades são infinitas e seu custo é acessível para diferentes tamanhos de equipe. O *Bonde do deploy* ainda não está a todo vapor, mas a equipe se mostrou ansiosa para colocar tudo nos trilhos.

Com tantas soluções no mercado para integração contínua, como <a href="https://jenkins.io/" target="_blank">Jenkins</a> ou <a href="https://www.atlassian.com/software/bamboo" target="_blank">Bamboo</a>, o Pipeline se destacou na nossa equipe pela sua simplicidade e compatibilidade.
