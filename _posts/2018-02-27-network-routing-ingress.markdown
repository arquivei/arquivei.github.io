---
layout: post
title: Roteando tráfego de rede no Kubernetes com Ingress
date: 2018-02-27
categories: blog
img: network-routing-ingress.png
tags: [Kubernetes, Networking, Routing]
author: luccas
brief: OBJETIVO BREVE
---

# O que é:

<p style='text-align: justify;'>
O Ingress é um novo recurso, inserido no Kubernetes em versão beta desde o Server 1.1, e agora está oficialmente na versão 1.9
Ele é uma maneira de abstrair um conjunto de serviços, semelhante a como um serviço abstrai um conjunto de Pods.
Utilizando um conjunto de regras baseada no hostname da request, ele é capaz de rotear tráfego inbound para serviços internos do cluster.
</p>


![network-routing-ingress_01]({{ site.baseUrl }}/assets/img/posts/network-routing-ingress_01.png){:class="img-fluid"}

# Por que é interessante ?

<p style='text-align: justify;'>
Aqui na Arquivei nós estávamos expondo sistemas no Kubernetes, como nossa aplicação do Arquivei, nossas APIs, ou outros sistemas que precisam de um endpoint na internet, usando um conjunto de serviços do tipo LoadBalancer. Esses serviços tem um endereço de IP externo, e utilizam uma das distribuições de Load Balancers reais das plataformas da AWS ou GoogleCloud junto do conjunto de regras fornecido, roteando requisições para um conjunto de pods definido à partir de labels.
</p>

<p style='text-align: justify;'>
Com o tempo, nossos sistemas foram crescendo, e com isso nossa prática ficou custosa. Os Load Balancers alocados tem custo por unidade (AWS) ou por regras de encaminhamento (GCloud), que acabam se acumulando ao passo que nossos sistemas crescem e precisam disponibilizar mais serviços em novos endpoints, sendo agravado ainda mais quando precisamos replicar essa infraestrutura para os times de Dev em múltiplos ambientes.
</p>

<p style='text-align: justify;'>
Aqui entra o Ingress. Utilizando apenas um endereço de IP exposto na internet, uma distribuição do Controller do Ingress (Nginx, HAProxy, GCE ...) e um conjunto de regras disponibilizado na forma do recurso do Ingress, conseguimos rotear as requisições que chegam neste endereço para serviços internos ao Cluster, como ClusterIPs na AWS e NodePorts no GCloud, reduzindo o número de Load Balancers utilizados.
</p>

<p style='text-align: justify;'>
Além disso, o Ingress é a única maneira nativa de realizar Load Balancing com terminação de TLS para requests HTTPS no GCloud.
</p>
# O Ingress e os Controllers

<p style='text-align: justify;'>
Um conjunto de regras do Ingress só funciona se no cluster já estiver disponível uma distribuição do Deployment que recebe o nome de Controller do Ingress. Existem diversas distribuições, sendo possível desenvolver o seu próprio caso os disponíveis não cumpram o desejado. Dentre os disponíveis, usamos o nativo do GCE, que é uma versão  rodando por baixo o do Nginx, que é o que usamos na AWS.
</p>

## Como configurar um Controller

A primeira coisa que precisamos portanto é um Controller. Na AWS, podemos usar o seguinte Deployment oficial, mantido pelo Kubernetes:

Configs do controller:
```
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml \
    | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml \
    | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml \
    | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml \
    | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml \
    | kubectl apply -f -
```

Controller:
```
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/without-rbac.yaml \
    | kubectl apply -f -
```

Para ver que o controller está rodando, use
```
$ kubectl get pods --namespace ingress-nginx
```
Exemplo de output:
```
NAME                                                    READY       STATUS    RESTARTS       AGE
default-http-backend-4031882202-n6c9r                   1/1         Running       0          1d
nginx-ingress-controller-3818767641-49d4b               1/1         Running       0          1d
nginx-ingress-controller-3818767641-668f8               1/1         Running       0          1d
nginx-ingress-controller-3818767641-8s4fq               1/1         Running       0          1d
```
<p style='text-align: justify;'>Este controller pode ser customizado, mas com essas configs básicas ele já funciona. O número de réplicas do pod afeta quantas requests ele é capaz de direcionar.
</p>

<p style='text-align: justify;'>Essa imagem do controller usa o Nginx para rotear requests. Por meio de argumentos passados no deployment do Controller, podemos indicar qual a classe de controller que estamos criando (senão teremos por default a classe “nginx”). Essa classe é usada para indicar no recurso ingress qual Deploy de Controllers deve gerenciar seu conjunto de regras. A config usada é a seguinte:
</p>

```yaml
(parte de cima do controller)
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ingress-nginx
  template:
    metadata:
      labels:
        app: ingress-nginx
      annotations:
        prometheus.io/port: '10254'
        prometheus.io/scrape: 'true' 
    spec:
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.9.0
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --annotations-prefix=nginx.ingress.kubernetes.io
            - --ingress-class=my-nginx-class  #<------- AQUI ESTÁ A CLASSE DESSE CONTROLLER
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
          - name: http
            containerPort: 80
          - name: https
            containerPort: 443
```
<p style='text-align: justify;'>O Default backend é um argumento necessário no Nginx, mas nem toda distribuição de controller usa. É um deploy à parte e um serviço que respondem apenas uma página 404 simples, que pode ser customizada.
</p>

<p style='text-align: justify;'>Depois de aplicar o deployment, o conjunto de pods fica procurando recursos ingress para gerenciar. O próximo passo é expor o Controller para a internet. Para isso, usamos um serviço do tipo LoadBalancer. No GCloud, ele é gerado automaticamente, caso estejam usando o Kubernetes como serviço fornecido e gerenciado pelo GCloud. Na AWS, precisamos criar ele explicitamente, usando um Service do Kubernetes. Segue um exemplo:
</p>

```yaml
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app: ingress-nginx
  annotations:
    # replace with the correct value of the generated certifcate in the AWS console
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-west-2:XXXXXXXX:certificate/XX-XX-XX-XX" 
    # the backend instances are HTTP
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    # Map port 443
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
    # Increase the ELB idle timeout to avoid issues with WebSockets or Server-Sent Events.
    service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: '3600'
spec:
  type: LoadBalancer
  selector:
    app: ingress-nginx
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: https
    port: 443
    targetPort: http
```
<p style='text-align: justify;'>Este serviço terá um IP externo. Vemos em spec.selector a label app: ingress-nginx. No deployment anterior, vemos que o controller tem essa label. Com isso, temos o load balancer roteando todas as requests para o deployment do Controller. As portas realizam o roteamento para a http sempre, de acordo com a config.
</p>
<p style='text-align: justify;'>Depois de aplicado, use o seguinte comando para ver o serviço rodando
</p>

```
$ kubectl get service --namespace ingress-nginx-dev
```
Exemplo output:
```
NAME                        CLUSTER-IP           EXTERNAL-IP            PORT(S)                       AGE
default-http-backend        100.67.66.75         <none>                 80/TCP                        51d
ingress-nginx-dev           100.68.123.176       c00510d80d917...       80:30728/TCP,443:30490/TCP    51d
```
<p style='text-align: justify;'>Como visto, temos o campo EXTERNAL-IP para nosso load balancer do ingress, que será usado como o endereço de IP para as requests que devem ser tratadas pelo controller.
</p>
<p style='text-align: justify;'>Nas annotations temos indicado o certificado que será usado para terminar o TLS e enviar a request http para o controller. Toda a terminação é feita diretamente no Load Balancer, mas esta opção só é possível na AWS.
</p>
<p style='text-align: justify;'>No caso do GCloud não temos a opção de terminar TLS no LB pois precisamos utilizar o Ingress para isso. (Não precisamos também realizar o Deploy do controller no GCloud, ele é gerenciado automaticamente.)
</p>

## Como configurar um Ingress.
<p style='text-align: justify;'>Um recurso Ingress é bem simples. Nele indicamos a classe do controller que queremos  realizando o link entre o controller e o Ingress. Depois disso, temos as regras de proxy para as requisições. Podemos indicar quais serviços e quais portas devem receber as requests de acordo com o host no header (aceita wildcard) e endpoints específicos, podendo um mesmo domínio ter diversos serviços diferentes sendo roteados por trás. Exemplo simples:
</p>

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    kubernetes.io/ingress.class: "my-nginx-class"
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /other_path
        backend:
          serviceName: s2
          servicePort: 80
  - host: my.website.com
    http:
      paths:
      - path: /*
        backend:
          serviceName: s3
          servicePort: 80

```
<p style='text-align: justify;'>Com esta configuração aplicada, o kubernetes gera o serviço do ingress e nossa annotation indica qual controller de Ingress deve cuidar dessas regras. Quaisquer requests que baterem no nosso serviço Load Balancer, que tiverem como host foo.bar.com no path /foo, ou seja, foo.bar.com/foo, serão direcionadas para o serviço s1 na porta 80. Porém o path /other_path leva as requisições para o serviço s2
</p>

<p style='text-align: justify;'>Caso o host seja my.website.com, o serviço s3 será responsável por essa request e como o path passado foi /* qualquer caminho será enviado para s3 e assim por diante. Quando uma request não tem um destino definido, ela recebe o default backend com uma página 404.
</p>
<p style='text-align: justify;'>Basta estender esta config para adicionar os seus serviços.
</p>
<p style='text-align: justify;'>Podemos montar diferentes Ingress para namespaces diferentes, sendo servidos pelo mesmo controller e mesmo load balancer. Um ingress só pode rotear requests para serviços no mesmo namespace. Para isso, basta aplicar as configs de ingresses nos namespaces desejados e manter nelas a mesma classe de controller indicada.
</p>

# Conclusão
<p style='text-align: justify;'>Usar o Ingress traz facilidades assim como desafios. Desde que implementamos em nossa infra, diminuimos o numero de Load Balancers que precisamos usar para manter todos os ambientes de 32 para 10 no cluster gerenciado pelo GCloud, e de 14 para 2 no cluster AWS de Prod e de 11 para 1 no de Dev.</p>
<p style='text-align: justify;'>
No total tivemos então uma redução de 57 Load Balancers para apenas 13, considerando clusters de Dev e Prod entre AWS e GCloud. Lembrando que não chegamos a usar todos esses Load Balancers na infra antiga, e sim que para replicar hoje a nossa infra da mesma maneira que usávamos a 2 mêses atrás teríamos estes numeros, mas realizamos a mudança antes que escalasse demais.
</p>
<p style='text-align: justify;'>
Ficou mais facil isolar os times e mais barato manter tudo rodando, mas ao mesmo tempo, adicionamos toda essa camada a mais de complexidade, o que pode complicar o debugging de um problema de rede não relacionado ao ingress, dada a falta de familiaridade, o quanto o recurso é recente e seu comportamento que difere dependendo da plataforma cloud usada.
</p>

<p style='text-align: justify;'>
</p>
## Considerações finais e cuidados
<p style='text-align: justify;'>Alguns últimos detalhes importantes para o uso do Kubernetes como serviço do GCP. Para que o Ingress realize o roteamento das requests para o seu serviço NodePort os pods referentes ao serviço precisam implementar uma ReadinessProbe com um endpoint que responda com código 200 a uma request https, e enquanto essa checagem não for efetiva o Ingress não vai realizar o roteamento das requests. Caso nenhuma ReadinessProbe seja implementada um healthcheck é feito automaticamente no serviço exposto.
</p>
<p style='text-align: justify;'>Sua ReadinessProbe pode ser um simples retorno de um container com apache ou nginx por exemplo, lembrando que o endpoint não pode terminar em “/” quando escrito no yaml, por exemplo “/health-check/”.
</p>
<p style='text-align: justify;'>A ReadinessProbe pode ser implementada também como um comando executado no container, que deve ter retorno 0 para ter sucesso, caso contrário o pod não fica Ready.
</p>
<p style='text-align: justify;'>Por último, temos o longo tempo de espera para que o Ingress comece a responder corretamente. Da nossa experiência, do início do nosso principal Deployment ter sido considerado Ready (Health check teve sucesso), até o Ingress deixar de responder com 404 ou 502, demora em torno de 10 minutos.
</p>

___

*Obs*
