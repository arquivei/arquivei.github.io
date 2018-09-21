---
layout: post
title: Subindo um cluster de kafka
date: 2018-09-03
categories: blog
img: kafka/kafka.png
tags: [Kafka, Zookeeper, Cluster, Fault Tolerance, Durability]
author: andre_missaglia
brief: Como subir um cluster de kafka em produção, de fácil manutenção, tolerante a falhas, e onde perder mensagens não é uma opção?
---
<style>
  .center {
    display: block;
    margin: 0 auto;
}
</style>

Rodar o kafka em um cluster é fácil: Basta baixar o código em cada uma das máquinas, configurá-las adequadamente, e iniciar o serviço. Mas como tornar este cluster tolerante à falhas, escalável, de fácil configuração, e garantindo que nenhum dado seja perdido? É possível se construir um cluster de kafka de múltiplas formas. Este artigo mostra uma maneira de abordar os pontos mencionados.


# Introdução

## O que é Kafka?

Kafka é um sistema de distribuição de mensagens que possui três grandes usos:

* **Sistema de mensagens**: Usado para desacoplar sistemas com escopos diferentes. 
* **Sistema de armazenamento**: Usado para guardar os eventos (logs) de forma consistente, permitindo reconstruir o estado do sistema a qualquer momento a partir destes logs. 
* **Processamento streaming**: Usado para transformar dados em tempo real, como mapeamentos, agregações, junções, etc. 

![log_consumer]({{site.baseurl}}/assets/img/posts/kafka/log_consumer.png)

Toda mensagem produzida recebe um *offset* sequencial, e fica armazenada em uma fila dentro do kafka. Cada consumidor é composto apenas por um ponteiro indicando a próxima mensagem a ser buscada. Mais detalhes de como funciona o kafka podem ser encontrados em sua documentação [[1]](https://kafka.apache.org/intro)

## O que compõe o kafka?

Para o kafka funcionar, é necessário que um zookeeper estaja rodando. Zookeeper é um serviço de armazenamento chave-valor, hierárquico e distribuído. Seu uso principal é para manter configurações e nomes consistentes em sistemas distribuídos. Por exemplo, o kafka consegue descobrir os outros brokers (nós) através do zookeeper.

![Arquitetura 1]({{site.baseurl}}/assets/img/posts/kafka/Architecture1.png){:class="center"}


# Montando um cluster de zookeeper

Se tratando de um cluster, e considerando que desejamos tolerância a falhas e fácil configuração, logo pensamos em instalar o zookeeper em uma instância EC2, criar uma imagem AMI a partir daquela instância, e criar um Auto Scaling Group que mantenha 3 instâncias do zookeeper no ar.

Porém temos que considerar o seguinte:

* O Zookeeper depende de uma configuração estática com o endereço de cada um dos nós no cluster
* O Auto Scaling Group não funciona com IPs fixos. Cada máquina ganha seu próprio IP aleatoriamente, o que dificulta a configuração.
* Para atualizar as configurações é necessário recriar as imagens AMI.

O que levou a adoção de:

* Exhibitor
* Pool de Elastic Network Interfaces (ENI)

Exhibitor [[2]](https://github.com/soabase/exhibitor) é um projeto, originalmente criado pela Netflix, que gerencia o Zookeeper. Ele permite manipular os dados armazenados, reiniciar o zookeeper em caso de falhas, monitorar logs e atualizar configurações dinamicamente.

As configurações ficam salvas em um bucket no S3, conforme detalhado neste artigo [[3]](https://jobs.zalando.com/tech/blog/rock-solid-kafka/?gh_src=4n3gxh1)

![Arquitetura 2]({{site.baseurl}}/assets/img/posts/kafka/Architecture2.png){:class="center"}

Além disso, o cada nó precisa ter um IP Fixo. Isso pode ser feito criando previamente interfaces de rede com IPs fixos. Por exemplo:

```
us-east-1a    10.0.10.10
us-east-1b    10.0.11.10
us-east-1c    10.0.12.10
```

Após isso, um script que roda na primeira vez que a instância é executada é responsável por buscar uma interface de rede disponível e ligá-la à instância.

```bash
# Busca interfaces de rede disponíveis, que possuam a label "Category:zookeeper"
export ENI=`aws ec2 describe-network-interfaces --filter Name=tag:Category,Values=Zookeeper Name=status,Values=available Name=availability-zone,Values=$AVAILABILITY_ZONE --query 'NetworkInterfaces[*].NetworkInterfaceId' --output text`; 

# Liga interface de rede à instância
aws ec2 attach-network-interface --network-interface-id $ENI --instance-id $INSTANCE_ID --device-index 1

# Altera rotas para usar a interface de rede nova
ip r change default dev eth1
ip r del `ip r | grep eth0`
```

Este script pode ser melhorado em alguns pontos antes de ser usado em produção:
* Procurar pela interface de rede por um número de vezes, se falhar, a instância deve ser removida.
* Aguardar a interface de rede estar funcional antes de utilizá-la

Com isso, basta criar uma AMI e iniciar o Auto Scaling Group. A instância, ao ser iniciada, buscará uma interface de rede disponível, irá ligar-se a ela, e iniciar o serviço do Exhibitor. Por sua vez, o Exhibitor buscará as configurações do S3, e iniciará o zookeeper.

# Montando um cluster de kafka
Agora temos a linha mais importante do arquivo de configuração do kafka:
```
zookeeper.connect=10.0.10.10:2181,10.0.11.10:2181,10.0.12.10:2181/kafka
```
E sabemos que esses IPs correspondem a um cluster de Zookeeper tolerante a falhas. Mas ainda falta duas configurações importantes `advertised.listeners` e `broker.id`.

## advertised.listeners
Para se conectar a um broker, a comunicação ocorrerá apenas através do endereço fornecido em `advertised.listeners`. Por exemplo, suponha que o kafka esteja configurado com `advertised.listeners=10.0.10.20`, mas que o cliente tente se conectar com o servidor através do endereço `node1.myhost.com`. Neste caso, o kafka *rejeitará* a conexão, avisando que o endereço `10.0.10.20` deve ser utilizado. O cliente então tentará se comunicar novamente com o kafka, mas desta vez através do endereço `10.0.10.20`.

Desta forma, *a escolha do `advertised.listeners` define quais clientes possuem acesso ao cluster*, uma vez que deve existir uma rota direta entre o cliente e o broker. Portanto, não é possível deixar o kafka em uma rede privada, com um load balancer visível para o cliente: o cliente deve poder se comunicar com o broker diretamente.

Este artigo descreve como montar um cluster privado. Ou seja, só é possível se comunicar com o kafka usando a rede interna. Então vamos usar o ip local da máquina:
```
advertised.listeners=PLAINTEXT://$LOCAL_IP:9092
```

Como o ip só será conhecido no momento que a máquina for iniciada, colocamos um placeholder `$LOCAL_IP`, que será substituído na primeira execução usando o comando `envsubst`.[[4]](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html)

Mas se o ip não é conhecido, e variável, como o cliente sabe como se conectar ao kafka? Uma solução seria usar um pool de interfaces de rede, assim como foi feito no zookeeper. Porém aqui há uma alternativa mais simples: Podemos usar um load balancer para redirecionar para uma das instâncias do kafka. Este Load Balancer possui apenas a função de Service Discovery, uma vez que a comunicação de fato ocorrerá diretamente entre o cliente e o broker. 

## broker.id

Outra configuração que precisa de atenção é o id de cada broker. Aqui, precisamos antes conhecer o funcionamento do kafka para entender os problemas de cada solução.

### Tópicos, partições, replicação, líder e eleição

Cada tópico é dividido em partições, que determinam o nível de paralelismo do lado do consumidor. Cada partição pode ser lida por no máximo um consumidor por grupo, ou seja, se um tópico possui 8 partições, pode haver no máximo 8 consumidores concorrendo pelo mesmo tópico (Caso pertençam ao mesmo grupo).

Cada partição é servida por apenas um broker, o líder, mas as mensagens são replicadas entre os outros brokers para tolerância a falhas. Caso o líder se torne inacessível, um de seus seguidores irá assumir a função de líder e passará a servir os clientes.

```bash
kafka-topics.sh --zookeeper zk_host:2181/kafka --topic mytopic --describe
# Result:
Topic:mytopic    PartitionCount:8        ReplicationFactor:3     Configs:
        Topic: mytopic   Partition: 0    Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: mytopic   Partition: 1    Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
        Topic: mytopic   Partition: 2    Leader: 2       Replicas: 2,1,3 Isr: 2,1,3
        Topic: mytopic   Partition: 3    Leader: 3       Replicas: 3,1,2 Isr: 3,1,2
        Topic: mytopic   Partition: 4    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3
        Topic: mytopic   Partition: 5    Leader: 2       Replicas: 2,3,1 Isr: 2,3,1
        Topic: mytopic   Partition: 6    Leader: 3       Replicas: 3,2,1 Isr: 3,2,1
        Topic: mytopic   Partition: 7    Leader: 1       Replicas: 1,3,2 Isr: 1,3,2
```
O comando acima mostra um cluster com 3 brokers, com um tópico de 8 partições e fator de replicação 3. Cada partição é servida por um broker diferente. Por exemplo, a partição 7 possui como líder o Broker 1. Caso este broker se torne indisponível, o próximo a assumir deve ser o broker 3, seguido do 2.

Este comando também mostra qual é o conjunto de "In Sync Replicas (ISR)". Como todos os 3 brokers aparecem em todas as partições, significa que todos os brokers estão devidamente sincronizados com o líder.

Agora vamos ver o que acontece em um cenário de falha:
```
Topic: mytopic   Partition: 7    Leader: 3       Replicas: 1,3,2 Isr: 3,2
```

Quando o broker 1 sai do ar, o broker 3 imediatamente assume a função de líder, e o 1 sai do conjunto Isr. Quando este broker voltar ao ar, o kafka irá sincronizar as mensagens, levando ao seguinte estado:
```
Topic: mytopic   Partition: 7    Leader: 3       Replicas: 1,3,2 Isr: 1.3,2
```

Perceba que, apesar do broker 1 estar de volta, ele não é o líder da partição. Ou seja, a partição 7 está operando sem o seu *líder preferido*. Isto pode ser um problema, pois alguns brokers podem ficar sobrecarregados enquanto outros ficam livres.

Para corrigir este problema, poderíamos rodar o comando:
```
kafka-preferred-replica-election.sh --zookeeper zk_host:2181/kafka
```

Ou então poderíamos deixar o kafka gerenciar isso automaticamente usando a configuração `auto.leader.rebalance.enable=true` (default).

### Primeira alternativa: não setar broker.id

Uma vez que o cluster está em um Auto Scaling Group, não é possível configurar ids fixos, uma vez que a mesma imagem será usada em todos os brokers. Porém o kafka possui a opção de setar esta configuração automaticamente.

Porém voltando ao exemplo anterior, teríamos a seguinte situação:
```
Topic: mytopic   Partition: 7    Leader: 1001       Replicas: 1001,1003,1002 Isr: 1001,1003,1002
```
Perceba que os IDs são gerados em um range diferente dos IDs configurados manualmente. Porém, quando o nó 1001 sair do ar, o próximo ID gerado será o 1004, o que levará à seguinte situação:
```
Topic: mytopic   Partition: 7    Leader: 1003       Replicas: 1001,1003,1002 Isr: 1003,1002
```

Ou seja, mesmo com um cluster com todos os 3 brokers funcionando corretamente, apenas 2 estarão servindo partições, enquanto o último ficará ocioso. Isso ocorre porque o broker 1004 não é réplica de nenhuma partição.

Este problema pode ser contornado usando a ferramenta de reatribuição de partições `kafka-reassign-partition.sh`, onde deve ser gerado um arquivo com as atribuições, para posteriormente o kafka executar estas atribuições. Detalhes desse processo podem ser encontrados em [[5]](https://blog.imaginea.com/how-to-rebalance-topics-in-kafka-cluster/).

Porém este processo é complexo, e não faz uso das ferramentas de recuperação de falhas do próprio kafka.

### Segunda alternativa: setar broker.id com base em algum critério

Considerando um cluster de 3 máquinas espalhadas em diferentes AZ, é possível obter um mapeamento do tipo:
```
us-east-1a   id=1
us-east-1b   id=2
us-east-1c   id=3
```

Porém essa solução limita o tamanho do cluster ao número de AZ, e não permite múltiplos brokers em uma mesma região. Claramente precisamos de algo mais escalável.

Portanto vamos criar um script que retorne um id, seguindo os seguintes passos:

1. Comece com um pool de ids, ex.: 1,2,3
2. Busque no zookeeper quais brokers estão ativos, ex.: 1,3
3. Subtraia os dois conjuntos, ficando com o conjunto de ids livres, ex.: 2
4. Retorne um id do conjunto restante.

Obs: Este pool não precisa ter o mesmo tamanho do cluster. É possível começar com conjunto de 100 ids por exemplo. Neste caso, o critério para escolha do id do conjunto de ids livres deve ser o menor deles. Assim, mesmo com um pool de 100 ids, se o cluster tem tamanho 3, apenas os ids 1,2,3 serão usados.

Porém este algorítmo possui uma falha: Caso dois brokers estejam subindo ao mesmo tempo, eles possuirão o mesmo id, e apenas um deles conseguirá subir com sucesso, enquanto o outro falhará.

Vamos resolver este problema de duas formas:
* Esperar um tempo aleatório antes de buscar o id no zookeeper. Isso diminui a probabilidade de dois nós buscarem o mesmo id.
* Usar o Health Check do load balancer. Se o kafka não conseguir subir por estar com o mesmo id que outro broker, a instância será marcada como "unhealthy", e será substituída por outra eventualmente pelo Auto Scaling Group.

Vale lembrar que este problema só acontece durante a primeira vez que o kafka for lançado. Após isso, dificilmente múltiplos nós irão reiniciar simultaneamente.

Por fim, temos o seguinte script:
```python
# getid.py
import os
import random
import time
from kazoo.client import KazooClient

time.sleep(random.uniform(0, 20))

zk = KazooClient(hosts=os.getenv("ZK_HOSTS"))
zk.start()

try:
  zk_broker_ids = zk.get_children('/kafka/brokers/ids')
  set_broker_ids = set(map(int, zk_broker_ids))
  possible_broker_ids = set(range(1,100))
  
  broker_id = sorted(possible_broker_ids - set_broker_ids)[0]
  
  print broker_id
except: 
  # Na primeira execução, não existe a chave /kafka no zookeeper
  print 1
```

Agora temos a última configuração:
```
broker.id=$BROKER_ID
```

Basta colocar no script que é executado quando o broker inicia pela primeira vez (no caso da AWS, user data) as seguintes linhas, para criar o arquivo de configurações definitivo:
```bash
export BROKER_ID=`python getid.py`
export LOCAL_IP=`ip addr show eth0 | grep -Po 'inet \K[\d.]+'`
envsubst '$LOCAL_IP,$BROKER_ID' < /kafka/app/config/server.properties.dist > /kafka/app/config/server.properties
```

# Segurança
Segurança no kafka é um assunto complexo. Há várias formas de se configurar, e não há uma solução que funcione para todos os casos. Aqui, vamos mostrar uma das possibilidades. Os tópicos são divididos entre **Criptografia**, **Autenticação** e **ACL**.

## Criptografia
O Kafka permite que as comunicações cliente-servidor e servidor-servidor sejam criptografadas usando SSL. Porém não vamos usar criptografia pelos seguintes motivos:

* Esta rede é interna, portanto, o risco de ter mensagens interceptadas é mínimo (E se algo invadir esta rede, teremos problemas maiores para lidar...).
* Criptografia gera um overhead de processamento, causando queda de performance.
* A configuração é relativamente complexa, exigindo criar uma autoridade certificadora reconhecida por todos os clientes, e que todos os brokers possuam certificados assinados.

## Autenticação

Autenticação tem como objetivo restringir quem tem acesso ao cluster, além de permitir o uso de regras de controle de acesso.
Existem dois protocolos de autenticação, que usam o framework SASL[[6]](https://en.wikipedia.org/wiki/Simple_Authentication_and_Security_Layer):

### SASL_SSL

Assim como o no SSL o servidor se "autentica" com o cliente através do certificado digital, provando que o servidor é ele mesmo, é possível que o cliente se autentique com o servidor caso ele também possua um certificado digital assinado.

### SASL_PLAINTEXT

Outro modo de se autenticar é através do protocolo *SASL_PLAINTEXT*, que não exige o uso de SSL. Um dos mecanismos deve ser usado.

#### SASL_PLAINTEXT: Mecanismo PLAIN
Consiste em enviar usuário/senha em texto plano. Note a diferença entre *PLAIN* e *PLAINTEXT*. *PLAIN* se refere ao mecanismo de autenticação, enquanto *PLAINTEXT* se refere à camada de transporte. Por motivos de segurança, *PLAIN* só deve ser usado caso a camada de transporte seja SSL.

#### SASL_PLAINTEXT: Mecanismo SCRAM
Protocolo baseado em chalenge-response[[7]](https://en.wikipedia.org/wiki/Salted_Challenge_Response_Authentication_Mechanism). Também consiste em um par usuário/senha, mas que não são trocados diretamente.

#### SASL_PLAINTEXT: Mecanismo GSSAPI (Kerberos)
Depende de um servidor externo (ex.: Active Directory) que controla quem são os usuários com acesso.

#### SASL_PLAINTEXT: Mecanismo OAUTHBEARER
Faz uso do *OAuth 2 Authorization Framework*, onde um outro servidor será usado para autenticar os clientes

### Implementando SASL/Scram
Pela segurança e relativa simplicidade, vamos implementar a autenticação usando o protocolo SASL_PLAINTEXT com mecanismo SCRAM.

```
# Usando listeners, e não advertised.listeners
# Trocando PLAINTEXT por SASL_PLAINTEXT
listeners=SASL_PLAINTEXT://$LOCAL_IP:9093 

# Especificando o mecanismo do SASL
sasl.enabled.mechanisms=SCRAM-SHA-256

# Habilita autenticação entre os brokers
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
```

Nas configurações acima especificamos que a comunicação entre os brokers será autenticada. Portanto, precisamos especificar as credenciais. Isso é feito da seguinte forma:

* Criar arquivo JAAS com as credenciais

/kafka/kafka_server_jaas.conf
```

KafkaServer {
   org.apache.kafka.common.security.scram.ScramLoginModule required
   username="admin"
   password="pass";
};
```

* Especificar este arquivo no script que inicia o kafka, adicionando a seguinte linha ao arquivo `kafka-server-start.sh`

```
export KAFKA_OPTS=-Djava.security.auth.login.config=/kafka/kafka_server_jaas.conf'
```

Por fim, é necessário criar o usuário `admin`, diretamente no zookeeper:
```
kafka-configs.sh --zookeeper zk_host:2181/kafka --alter --add-config 'SCRAM-SHA-256=[password=pass],SCRAM-SHA-512=[password=pass]' --entity-type users --entity-name admin
```

## ACL

Por fim, podemos habilitar no kafka o controle de acesso para cada usuário. Informações mais detalhadas sobre como criar tais regras podem ser encontradas em [[8]](https://docs.confluent.io/current/kafka/authorization.html)

As seguintes configurações precisam ser setadas:
```
# Classe padrão de autorização
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer

# Se não houver regra, o usuário será negado
allow.everyone.if.no.acl.found=false

# Status de superusuário para o admin, usado para autenticação entre brokers
super.users=User:admin
```

# Conclusão

O processo para executar tanto o zookeeper quanto o kafka é relativamente simples: 

1. Criar arquivo de configuração
2. Iniciar o serviço

Porém como vimos, há alguns truques para atingir os objetivos desejados (tolerância à falhas, fácil configuração, etc.). No caso do zookeeper, usamos tanto o Exhibitor para gerenciar as duas etapas quanto um pool de interfaces de rede para manter os ips fixos. Para o kafka, precisamos nos atentar a duas configuraçõs: `broker.id` e `advertised.listeners`. Para o primeiro, usamos um script que busca um id livre no zookeeper. Para o segundo, usamos o próprio ip local da máquina.

Além disso, usamos alguns recursos da Amazon, como Auto Scaling Group, Load Balancer, e S3. Por fim, chegamos à seguinte arquitetura:

![Arquitetura 3]({{site.baseurl}}/assets/img/posts/kafka/Architecture3.png){:class="center"}

Vale ressaltar que esta arquitetura foi desenvolvida com um conjunto de objetivos em mente. Para outros casos de uso, como por exemplo, se fôssemos deixar o cluster público, precisaríamos nos atentar a outros detalhes, como criptografia SSL.

Ainda há muito o que pode ser feito para melhorar a manutenção e adicionar funcionalidades ao kafka, por exemplo:

* Kafka Connect
* Schema Registry
* REST Proxy
* Coletar métricas (JMX, Datadog, etc.)
* Usar ferramentas de manutenção (Kafka Manager, Topics UI, etc.)

Mas todas essas ferramentas possuem como requisito um cluster funcional de kafka, o que conseguimos com as técnicas deste artigo.