---
layout: post
title: A jornada pela integração contínua - Parte 1
date: 2018-03-14
categories: blog
img: bitbucket-pipeline.png
tags: [Automation, Bitbucket, Docker, gcloud]
author: flipper
brief: O sonho de todo desenvolvedor é apertar um botão e ver o código em produção. Aqui conto um pouco da jornada que tive acelerando o processo de build/deploy da equipe de Frontend da Arquivei.
---
<style>
  .wrap-content p {
    text-align: justify;
    text-indent: 25px;
  }

  .wrap-content a {
    text-decoration: none;
  }

  pre {
    background-color: #2d2d2d;
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
</style>

<center><i>Continuous Integration. Porque quem faz o build, tem pressa.</i></center>

Antes de falar da jornada é relevante falar da motivação. A integração contínua, deploy contínuo ou o *Bonde do deploy* como a gente chama por aqui, é um tópico recorrente nas conversas da Engenharia. A mentalidade "Release early, release often" é um consenso da equipe, mas por conta da arquitetura atual ainda não é a regra. Muitos buscam de alguma forma facilitar o processo de deploy, e a equipe de SRE trabalhou bastante para deixar a vida dos desenvolvedores mais fácil. 

Nesta série de artigos vou contar um pouco do que eu consegui fazer para deixar a vida deles, SREs, mais fácil (e por tabela a nossa mais fácil ainda).

# O processo

Usamos o modelo de imagens e contêineres, com a ferramenta <a href="https://www.docker.com/" target="_blank">Docker</a>, para gerar nossas versões de teste e produção. Gerar uma imagem é tão simples quanto rodar <span class="code">$ docker build -t frontend .</span> no diretório do seu projeto (já com o Dockerfile pronto). Para fazer um deploy, tanto em produção quanto no ambiente de testes, basta trocar a imagem base do frontend. É comum ouvir de um colega do time de QA *"Me vê aquela imagem pra testar aí"*.

<small>Esta série de artigos tem como foco a integração contínua, logo a stack do Frontend não será discutida por ser "irrelevante" para o processo. Caso a stack fosse diferente o processo seria extremamente similar, senão o mesmo.</small>

O desenvolvimento é feito numa branch do desenvolvedor, e uma vez que a funcionalidade está pronta para testes é gerada uma imagem que será colocada num dos ambientes de teste. Após ser testada, é feito um pull request para a revisão de código e a funcionalidade entra na branch <span class="code">master</span>. A versão de produção sempre é criada a partir da <span class="code">master</span>, sem uma rotina pré determinada. O deploy sobe quando tem que subir.

Após a imagem ser criada ela recebe uma tag e é enviada para o Google Container Registry <a href="https://cloud.google.com/container-registry/" targe="_blank">(GCR)</a> e para o Amazon Elastic Container Registry <a href="https://aws.amazon.com/pt/ecr/" target="_blank">(ECR)</a>, onde ficam armazenadas.

# O problema

Esse modelo é relativamente simples, mas alguns detalhes são importantes para entender os reais problemas enfrentados:

* **Gerar uma imagem é custoso:** A stack do Frontend não é exatamente "leve" e o node não é exatamente conhecido por sua performance. O build final que vai para o cliente é bem leve, mas o processo de geração da imagem pode demorar alguns minutos.
* **Tempo é dinheiro:** O build da imagem consome uma boa quantidade de recursos da máquina. Antigamente o build das imagens de teste era feito na máquina do desenvolvedor, o que era uma ótima desculpa pra ir tomar um café por 15 minutos ~~ou mais~~.
* **Teste é responsabilidade do desenvolvedor:** A equipe de SRE é responsável pelo deploy de produção. As imagens e o deploy nos ambientes de teste são de responsabilidade do desenvolvedor e do QA responsável. Sendo assim é fundamental acelerar esse processo já que não vamos importunar os SREs com essas tarefas.

Com vários testes realizados pelo QA por semana fica evidente que muitas horas eram gastas pela equipe de Frontend gerando imagens de teste. Além disso o deploy de produção fica mais lento já que temos que fazer um pedido toda vez que julgamos necessária uma nova release. Pensando nisso tomei a iniciativa de automatizar o processo de geração de imagens.

# A solução inicial

O primeiro problema a ser atacado era o tempo perdido gerando imagens nas nossas máquinas. Com isso em mente fui em busca de soluções na cloud para delegar essa tarefa a outro sistema, assim ficaríamos livres para continuar com o trabalho durante a geração das imagens de teste. Alguns dias de estudo deram soluções boas e soluções ruins, e ao final decidi criar uma instância no <a href="https://cloud.google.com/products/compute/" target="_blank">Gcloud Compute</a>.

O problema principal dessa instância é o seu preço. Não seria viável deixar a instância ligada durante o dia só esperando as requisições de build, então tive que criar um script para iniciar a máquina ao fazer uma requisição e desligá-la caso nenhuma nova requisição chegasse após x minutos. Após duas semanas de tentativas e erros cheguei na versão final do meu script <span class="code">publish.sh</span>.

```sh
##################################################
# Faz o build remoto a partir do último commit
# usage: usage: sh publish.sh -s <OAuthKey> -k <OAuthSecret>
##################################################

while getopts k:s:t:e:R opt; do
  case "${opt}" in
    k) oauth_key=${OPTARG};;
    s) oauth_secret=${OPTARG};;
    t) tag=${OPTARG};;
    e) env_file=${OPTARG};;
    R) replace_env="true";;
  esac
done

if [ -z $oauth_key ] || [ -z $oauth_secret ] ; then
  echo "Missing OAuth credentials"
  echo "usage: sh publish.sh -s <OAuthKey> -k <OAuthSecret>"
  exit 1
fi

echo "Fetching OAuth access token..."
# Recupera o Access Token como credencial do git
token=$(curl https://bitbucket.org/site/oauth2/access_token -d grant_type=client_credentials -u $oauth_key:$oauth_secret | grep access_token | cut -d ':' -f2 | cut -d '"' -f2)
# Termina a execução se a recuperação do token falhar
if [ -z $token ] ; then
  echo -e "\\e[0;41m[TOKEN ERROR]:\\e[0m\033[0;31m Could not fetch the OAuth token. Try again in a few minutes.\033[0m"
  exit 1
fi

# Inicia a instância se estiver desligada
# status=$(gcloud compute instances list | grep <nome_da_instancia> | tr -s ' ' | cut -s -d ' ' -f6)
# if [[ $status != "RUNNING" ]] ; then
gcloud compute instances start <nome_da_instancia> --zone=us-central1-c
# else
#   echo "Instance is already running, skipping boot..."
# fi

# # Recupera o último commit remoto da branch atual se nenhuma tag for passada
if [ -z $tag ] ; then
  tag=$(git rev-list HEAD --max-count=1 --abbrev-commit)
fi

# # Caso não seja passado um .env como parâmetro irá usar o arquivo remoto da máquina
if [ -z $env_file ] ; then
  env_command="cp .env"
else
  env_file64=$(base64 $env_file)
  env_command="echo \"$env_file64\" | base64 -d >"
fi

if [[ $replace_env = "true" && -n $env_file ]] ; then
  replace_env_command="echo \"$env_file64\" | base64 -d > ~/.env;"
else
  replace_env_command=""
fi

# Limpa pastas e chamadas de shutdown
clearBuild="sudo shutdown -c; sudo rm -rf /root/frontend_$tag;"
# Clona o repositório e faz o checkout no commit recebido como parâmetro
prepareBuild="cd ~; git clone https://x-token-auth:$token@bitbucket.org/<repositorio_frontend> frontend_$tag; $replace_env_command $env_command frontend_$tag/.env; cd frontend_$tag; git checkout $tag;"

# Faz o build dos arquivos, gerando a imagem do docker
build="timeout 10m make build TAG=$tag || ([ $? -eq 124 ] && echo \"\\e[0;41m[TIMEOUT] PROCESS TERMINATED:\\e[0m\033[0;31m Time limit exceeded. Program ran for more than 10minutes. Try again.\033[0m\");"
# Sobe os arquivos para a AWS
publishAWS="export PATH=~/.local/bin:$PATH;\$(aws ecr get-login --region <regiao> --no-include-email);docker push <ecr_url>/frontend:$tag;"
# Sobe os arquivos para o GCR
publishGCR="sudo /usr/bin/gcloud docker -- push <gcr_url>/frontend:$tag;"
# Limpa as imagens dangling e todas as imagens criadas 4h atrás ou mais
clearDangling="docker rmi \$(docker images -f \"dangling=true\" -q) -f || true;"
clearOldImages="docker rmi \$(docker images | grep \"\([1-9][0-9]\|[4-9]\) hour\|day\|week\|month\" | tr -s \" \" | cut -s -d \" \" -f3) -f || true;"
echo "Connecting..."

# Conecta na VM e passa os comandos para execução remota
gcloud compute --project <projeto> ssh <nome_da_instancia> --zone <regiao> --command \
    "sudo su -c \
    '
      $clearBuild
      $clearDangling
      $clearOldImages
      $prepareBuild
      $build
      $publishAWS
      $publishGCR
      $clearBuild
    ';
    sudo shutdown -P +15"

exit 0

# Caso o build demore mais de 12 minutos a execução é interrompida. Execute novamente ou faça manualmente.
# A máquina irá desligar em 15 minutos após o build e o publish serem finalizados. Caso outro build entre em execução a contagem é reiniciada.
```

<small>Removi alguns detalhes privados como nome da instância, url do repositório e urls da Amazon e Google.</small>

O script é invocado usando o seguinte comando no diretório do repositório: <br/><span class="code">$ sh publish.sh -k \<OAuthKey\> -s \<OAuthSecret\> [-t Tag] [-e EnvFile] [-R]</span>
<small><br/>
As credenciais OAuth são configuradas individualmente por cada desenvolvedor no Bitbucket e permitem que o repositório seja clonado sem a necessidade do envio de uma chave privada.
</small>

O processo de desenvolvimento ficou muito mais rápido e simples. Agora bastava colocar o comando acima num alias e executar toda vez que fosse necessário gerar uma imagem de testes. A imagem era criada remotamente e em menos de 12 minutos estava pronta para ser usada pelo QA.

Tudo parecia muito bom e a equipe de Frontend estava satisfeita com o resultado. Agora todos tinham mais tempo útil já que os não precisavam ficar esperando o build das imagens de testes. Mas isso foi só uma melhoria interna, não refletia na produtividade da Engenharia como um todo e não era escalável para o uso em produção. Eu ainda queria que tudo fosse mais integrado. Com o primeiro obstáculo fora do caminho era hora de trabalhar no *Bonde do Deploy*.

# A solução final

Precisávamos de uma ferramenta escalável e de preferência com suporte a diferentes tecnologias para abranger toda a stack da Engenharia. Além disso seria melhor usar uma ferramenta que já conhecida, para evitar uma barreira de complexidade ao fazer uma migração para ela. Não muito tempo depois descobri que uma ferramenta que já utilizávamos poderia ser a solução para todos os problemas descritos nesse artigo. Foi aí que usei o Pipeline para criar nossas imagens, tanto de teste como produção, de forma integrada ao desenvolvimento.

Vou contar o processo de criação dessa solução na Parte 2 desse artigo.