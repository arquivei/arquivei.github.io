---
layout: post
title: Processando eventos genéricos em streaming usando BigQuery e Dataflow
date: 2018-12-07
categories: blog
img: genericpipeline/main-picture.jpg
tags: [Apache Beam, Spotify SCIO, Scala, Google Dataflow, BigData, Stream Processing, Mutating Events, Google PubSub, Google BigQuery, Apache Kafka]
author: eduardo_soldera
brief: "Construindo um pipeline de dados que cria e evolui tabelas automaticamente no Google BigQuery a partir dos dados de eventos chegando em streaming, usando um wrapper para Apache Beam em Scala executado no Google Dataflow."
---
<style>
  .center {
    display: block;
    margin: 0 auto;
}
</style>

Pipelines de dados em streaming são muito utilizados para consumir eventos de uma fila e disponibilizá-los em um Data Warehouse para consulta. Nesse artigo, apresentamos uma solução de pipeline que lê de um stream de eventos e realiza uma inserção dinâmica em tabelas do BigQuery a partir dos dados contidos em cada evento, sem nos preocuparmos em definirmos o schema de cada evento no código do pipeline. 
Utilizamos uma API do Spotify (Scio) em Scala, que encapsula as bibliotecas do Apache Beam, uma implementação do modelo de programação Dataflow, criado pelo Google, voltado para processamento de dados em streaming e batch, que executa no Google Dataflow e em outros runners de processamento de dados, como o Apache Spark e Flink.


# Contexto

Na Arquivei, utilizamos a arquitetura baseada em eventos para análises sobre os dados que chegam da nossa aplicação, integrações com outras plataformas e entrega de painéis de saúde de negócio. 
Possuímos centenas de eventos por segundo chegando via streaming. O protocolo utilizado é o JSON e temos em torno de uma centena de Schemas diferentes, com esse número crescendo a cada dia. <br>
O fato de possuirmos centenas de eventos diferentes trouxe um problema de produtividade e escalabilidade para o time de dados, porque para cada novo tipo de evento ou campo, existia a necessidade de fazer uma alteração no código e novo deploy nos pipelines de dados existentes.<br>
Mesmo com a existência de diversas soluções automatizadas de testes e deploy, a necessidade de uma alteração em código envolve a escrita e execução de testes, revisão de código e alterações no versionamento, além de uma migração de Schema das tabelas de destino dos eventos.<br>
Na prática, tratar um simples campo extra de um JSON podia levar algumas horas. Se a equipe que estava alterando o evento não avisava o time de dados com antecedência, os dados extras eram perdidos até um reprocessamento do dado ser executado.<br>
Resumo: dor de cabeça.

## Nossa stack

Como Data Warehouse, utilizamos o [BigQuery](https://cloud.google.com/bigquery/), altamente escalável, barato, integrável com uma série de ferramentas de Business Intelligence e completamente gerenciado pelo Google Cloud Platform. Claro que tudo depende do caso de uso, porém já vimos empresas perdendo <b>muito</b> tempo partindo para soluções Hadoop para executar queries em dados estruturados. 

Para o desenvolvimento de Pipelines, utilizamos o [Apache Beam](https://beam.apache.org), projeto open source originado de códigos fechados do Google, propõe um modelo de programação voltado para processamento de dados em streaming e batch de forma unificada. O Beam disponibiliza bibliotecas em Java, Python e Go. O Spotify mantém o [Scio](https://github.com/spotify/scio), um projeto open source que encapsula as bibliotecas Java do Beam, oferecendo uma interface na linguagem [Scala](https://www.scala-lang.org/), que é interoperável com Java e combina orientação a objetos com programação funcional.

Como estávamos utilizando as bibliotecas Java de Apache Beam, migramos os pipelines para Scala com o Scio, para melhorar a velocidade na entrega dos códigos, legibilidade <strike>(era Java, eca)</strike> e para utilizarmos dos recursos sensacionais que o Scala oferece. 

Como infraestrutura de mensageria, utilizamos o Apache Kafka, porém qualquer serviço de mensageria em Streaming que tenha uma source facilmente instanciável no Apache Beam, como o Google Pubsub, pode ser utilizado sem dores de cabeça.

Por fim, como runner dos pipelines do Apache Beam, utilizamos o [Dataflow](https://cloud.google.com/dataflow/?hl=pt-br), também gerenciado pelo Google. Apesar de existir quase [uma dezena de maneiras de rodar pipelines do Apache Beam](https://beam.apache.org/documentation/runners/capability-matrix/). O Dataflow tem uma série de garantias e facilidades por ser gerenciado, como integração com o Google Stackdriver, retentativas em caso de falhas, interface mostrando o estado atual de processamento dos pipelines, auto escalabilidade e outros.

## Formato dos JSONs

Paralelamente à questão abordada, o time de Engenharia de Dados havia recentemente imposto uma nova padronização de formato dos eventos, em que todos os produtores de evento deveriam seguir um envelope para os JSONs enviados:

```json

{
    "SchemaVersion": 1,
    "ID": "a-random-id",
    "Source": "event-source",
    "Type": "event-type",
    "CreatedAt": "2017-01-01T23:24:25.123456789-03:00",
    "DataVersion": 1,
    "Data": {

    }
}
```
Nesse envelope, existe uma informação de versão ```SchemaVersion```, que iria ser incrementada quando o envelope como um todo evoluísse. Analogamente, o ```DataVersion``` corresponde ao versionamento de evolução dos dados no campo ```Data```, que carregará os dados específicos de cada evento. 

Os campos ```Source``` e ```Type``` representam respectivamente o sistema de origem e o tipo do evento. O ```ID``` é o ULID (identificador único ordenável) gerado na produção do evento.
O campo ```CreatedAt``` é um campo que representa o momento de criação de evento e deve seguir o formato RFC3339.

A definição de um envelope padrão já facilitou a tradução dos eventos para tabelas do BigQuery, pois apenas os campos dentro do ```Data``` podem ter um parsing personalizado.


## Primeira tentativa de solução

Após a primeira padronização do envelope de eventos, tentamos a seguinte abordagem:

- Eventos que faziam tracking de usabilidade da plataforma ganharam um campo ```IsTracking = true``` e todos cairiam na mesma tabela, com o campo ```Data``` todo convertido em String. Abaixo o Schema dessa tabela:

```json
[
  {
    "name": "ID",
    "type": "STRING",
    "mode": "NULLABLE"

  },
  {
    "name": "Source",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "Type",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "SchemaVersion",
    "type": "INTEGER",
    "mode": "NULLABLE"
  },
  {
    "name": "DataVersion",
    "type": "INTEGER",
    "mode": "NULLABLE"
  },
  {
    "name": "Data",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "CreatedAt",
    "type": "TIMESTAMP",
    "mode": "NULLABLE"
  }
]
```

- Demais eventos tinham como destino uma tabela específica para cada evento, não necessariamente no mesmo dataset

Essa abordagem acabou gerando dois problemas:

- Tabelas espalhadas em datasets
- Para tirar dados de um JSON dentro de um campo String no BigQuery, trazia a obrigatoriedade de usar ou uma [User Defined Function (UDF)](https://cloud.google.com/bigquery/docs/reference/standard-sql/user-defined-functions?hl=pt-br) ou [funções de JSON em SQL](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#json-functions) o que acabava criando overhead sempre que qualquer não-desenvolvedor precisava 
executar uma query.

## E agora?

Começamos a buscar várias alternativas para resolver esse problema de escalabilidade e de produtividade. Precisávamos de uma solução que nos ajudasse a organizar as tabelas dentro BigQuery, para que pudéssemos manter todos os eventos no mesmo dataset. A nova solução também deveria acabar com qualquer necessidade de desenvolvimento e deploy para cada evento novo. Também estávamos buscando algo nos auxiliasse na evolução dos JSONs, o que era bem comum. Essa evolução dos Schemas acontecia sempre quando algum sistema ganhava uma feature nova ou a necessidade de acompanhamento de métricas era modificada/aumentava.

Pesquisamos várias alternativas de tecnologias e abordagens para atacar essa questão, até que encontramos [um artigo no blog do Google](https://cloud.google.com/blog/products/gcp/how-to-handle-mutating-json-schemas-in-a-streaming-pipeline-with-square-enix/) descrevendo um problema muito semelhante ao nosso, apresentando como solução as novas bibliotecas do Apache Beam para BigQuery, porém sem apresentar exemplos de código de forma mais detalhada. Utilizando a ideia apresentada nesse artigo, decidimos criar esse post para apresentar a nossa implementação da resolução desse problema.

# O "pipeline genérico"

Carinhosamente apelidado pelo time de Engenharia de Dados de "Pipeline genérico", começamos a desenvolver um pipeline para substituir quilos de código por apenas alguns métodos mágicos que iriam abstrair o conceito de parsear eventos e inseri-los no BigQuery.

A ideia geral do pipeline segue esse fluxo apresentado no artigo do Google:

![genericpipe-flow]({{site.baseurl}}/assets/img/posts/genericpipeline/mutating-json-2a2es.max-900x900.PNG){:class="center"}
<center>Retirado do Blog do Google Cloud</center>

Primeiramente, fazemos a leitura do sistema de mensageria (Apache Kafka/Google Pubsub). A mensagem então é parseada de String para sua representação de JSON, diferente conforme as libs que cada linguagem oferece.

Entre a etapa de parsing e de inserção, as particuliaridades dos formatos de evento deverão ser tratadas. Itens como padronização dos casings dos campos do JSON, formatos de data e fuso horário, conversões e inserção de dados enriquecidos, como hora de processamento, por exemplo.

Nesse momento (e em muitos outros), o [Pattern Matching de Scala](https://docs.scala-lang.org/tour/pattern-matching.html), elemento comum em linguagens funcionais, foi nosso melhor amigo.

Com os dados tratados, é hora de (tentar) inseri-los no BigQuery, utilizando sua biblioteca de IO, empoderada pela classe ```DynamicDestinations``` que permite a determinação da tabela de destino em tempo de execução. Se a tabela não existe, essa classe nos permite implementar uma lógica para determinação do Schema. A partir dessa determinação de Schema, uma tabela será criada com o Schema recém determinado a partir dos dados do evento. Caso a tabela de destino já exista, fazemos a tentativa de inserção do evento na tabela.

Ambos os casos podem falhar. Caso uma falha não intermitente ocorra, os eventos passam para a próxima etapa.

Com uma coleção de dados representando as falhas, comparamos o Schema da tabela atual do BigQuery com o do evento que teve falha na inserção. Fazemos um merge dos dois Schemas e tentamos uma mutação da tabela de destino. Se houve sucesso, tentamos inserir o evento novamente. Se houve uma falha ou se a nova inserção falhou, adicionamos o evento em uma tabela de fallback.


## Condições

Precisamos traçar algumas condições para o funcionamento com perfeição do nosso "Pipeline genérico". Estamos utilizando o JSON como protocolo de transmissão de dados. Essas condições poderiam ser garantidos com a mudança para um formato como o [AVRO](https://avro.apache.org/), que dispararia uma exceção no produtor de eventos caso ele não esteja produzindo no padrão esperado. Isso pode ser alcançado com a implantação do [Schema Registry](https://github.com/confluentinc/schema-registry) para o Apache Kafka e esse é o próximo passo que queremos dar para colaborar nos esforços de escalabilidade e padronização.

Atualmente as condições são:
- O evento é um JSON válido
- O evento segue o nosso envelope padrão (como apresentado no início do post)
- Um evento nunca terá um tipo de dado alterado em algum campo existente (exemplo de problema: ```{"AccountId": 1}``` -> ```{"AccountId": '1'}```)

Caso uma das condições seja quebrada (como já foi), os eventos serão inseridos na tabela de fallback.

# Resultados

Tivemos uma certa dificuldade em implementar esse pipeline pois todos do time estavam aprendendo Scala há muito pouco tempo, então houve um período de adaptação à lógica empregada e também ao domínio do Pattern Matching. As funções para inferir Schema e para merge de Schemas se apresentaram como um ótimo desafio, pois como o Schema do BigQuery e nossos eventos podem ter campos repetidos e aninhados, tivemos que implementar vários métodos recursivos para esse tratamento. Tivemos que nos aprofundar bastante nos detalhes do Apache Beam, da interoperabilidade Java <--> Scala e testar extensivamente todas as lógicas de obtenção de Schema a partir de testes com milhões de eventos.

A partir do momento de mutação de tabelas, o BigQuery precisa de alguns segundos para propagação da definição do novo Schema antes da tentativa de inserção, o que precisou ser tratado do nosso lado.

Quando colocamos o pipeline no ar, reprocessamos os eventos do passado (centenas de milhões de eventos) para criar apenas um dataset com as tabelas de eventos, para que pudéssemos organizar melhor nossa utilização do BigQuery. Mesmo com os problemas citados, tivemos pouquíssimos casos de eventos (menos de 0.005%) que acabaram caindo na nossa tabela de fallback, que não possui todos os campos parseados.

Atualmente, quando há uma alteração evolutiva ou eventos novos, não precisamos escrever uma linha de código sequer para tratar uma criação de tabela nova ou adaptação das tabelas existentes.

O ganho de produtividade foi enorme e possibilitou ao time o foco em novos projetos que trazem mais valor do que as alterações quase que mecânicas que eram realizadas anteriormente.
Com a versatilidade do auto scaling do Dataflow, não temos que nos preocupar com qualquer aumento abrupto ou permanente no volume de eventos. 

Qualquer mudança no envelope de eventos ou no sistema de mensageria requer uma adaptação no código, que pode ser feita de maneira rápida. O código pode ser adaptado para qualquer stream de eventos que utilizamos por aqui, além de fornecer módulos que podem ser compartilhados entre outros projetos com outras finalidades.

Abaixo vamos entrar em detalhes dos códigos e das implementações realizadas. Caso você tenha alguma dúvida ou quiser conversar conosco é só mandar um e-mail :) 


# Extra: Show me the code

A Seção abaixo é um apêndice com os detalhes de implementação do pipeline

## Como era antes

Antes de implementarmos o novo pipeline, cada evento precisava de um trecho de código de parse personalizado.

Abaixo o exemplo de um evento de login:

```json

{
    "SchemaVersion": 1,
    "ID": "148174891728947198492187",
    "Source": "app",
    "Type": "user-was-authenticated",
    "CreatedAt": "2017-01-01T23:24:25.123456789-03:00",
    "DataVersion": 1,
    "Data": {
        "IpAddress": "100.100.100.100",
        "UserAgent": "Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:47.0) Gecko/20100101 Firefox/47.0",
        "AccountID": 1,
        "UserID": 1,
        "UserEmail": "eduardo.soldera@arquivei.com.br"
    }
}
```


Abaixo está um exemplo de código que realizava o parse de um evento de Login.

```java
private static class LoginParser extends DoFn<TableRow, TableRow> {
    @ProcessElement
    public void processElement(ProcessContext context) throws UnsupportedEncodingException {
        TableRow inputTableRow = context.element();
        JSONObject inputJson = new JSONObject(inputTableRow);

        JSONObject data = inputJson.getJSONObject("Data");
        JSONObject user = data.getJSONObject("User");
        String eventType = inputJson.getString("Type");
        int dataVersion = inputJson.getInt("DataVersion");

        TableRow outputTableRow = new TableRow();
        outputTableRow.set("Source", inputJson.getString("Source"));
        outputTableRow.set("Type", eventType);
        outputTableRow.set("SchemaVersion", inputJson.getInt("SchemaVersion"));
        outputTableRow.set("DataVersion", dataVersion);
        outputTableRow.set("ID", inputJson.getString("ID"));
        outputTableRow.set("CreatedAt", inputJson.optString("CreatedAt"));
        outputTableRow.set("IpAddress", data.optString("IpAddress"));
        outputTableRow.set("UserAgent", data.optString("UserAgent"));
        outputTableRow.set("AccountID", data.optBigInteger("AccountID", DefaultValues.UNKNOWN_KEY));
        outputTableRow.set("UserID", user.optBigInteger("UserID", DefaultValues.UNKNOWN_KEY));
        outputTableRow.set("UserEmail", user.optString("Email"));

        LOG.info("parser output: " + outputTableRow);
        context.output(outputTableRow);
    }
}
```
No modelo de programação do Apache Beam, deve-se implementar a classe ```DoFn```, que será distribuída para processamento paralelo.<br>
De maneira simplista, uma função idempotente ```processElement``` realiza uma leitura (```context.element()```) e uma escrita (```context.output```) do dado processado.
Nesse caso o tipo de dado utilizado é o ```TableRow```, que representa uma linha de dados no BigQuery.

## Nova implementação

O código será apresentado na mesma sequência em que a lógica do pipeline foi apresentada (Tópico "Pipeline genérico")

Primeiramente, a leitura do sistema de mensageria é realizada, obtendo uma coleção de Strings ```SCollection na implementação do Scio e PCollection no Apache Beam do Java``` 

Com os eventos chegando como String, nós realizamos a conversão de String para o ```JValue```, o objeto JSON do [JSON4s](http://json4s.org/), que é uma representação AST (Abstract Syntax Tree) para o JSON em Scala. Os códigos que serão apresentados utilizarão essa lib.

Esse método e vários outros, utilizarão do [Pattern Matching de Scala](https://docs.scala-lang.org/tour/pattern-matching.html), como comentado anteriormente:

```scala
  def parseMessage(msg: String): JValue = {
    val formats = org.json4s.DefaultFormats
    Try(parse(msg)) match {
      case Success(obj) => obj
      case Failure(_) =>
        LOG.error(s"Problem parsing message! It may not be a valid message: $msg")
        JNull
    }
  }
```
Nesse momento, um erro é adicionado no LOG caso a string não siga o padrão JSON. Como utilizamos o Google Dataflow, criamos um aviso via [Stackdriver](https://cloud.google.com/stackdriver/), que nos envia um e-mail caso um erro chegue ao LOG.

Esse é o momento de realizar as adaptações citadas (padronização dos casings dos campos do JSON, formatos de data e fuso horário, conversões e inserção de dados enriquecidos, como hora de processamento). Como essa etapa é muito particular para cada caso, não entraremos em detalhes.

Na posse do JValue, o mesmo deve ser convertido para ```TableRow```, que é o tipo da lib do BigQuery que representa uma linha numa tabela do BigQuery, então nós precisamos de um método para realizar a conversão entre ```JValue``` (o objeto JSON para o JSON4s) e ```TableRow```.

```scala
object JsonToTablerow {
  def apply(input: JObject): TableRow = {
    val format = org.json4s.DefaultFormats
    val sanitizedInput = input.noNulls remove { //garante ausência de elementos nulos
      case JArray(listOf) => listOf match {
        case Nil => true
        case _ => false
      }
      case JObject(Nil) => true
      case _ => false
    }
    convert(sanitizedInput.noNulls).asInstanceOf[TableRow]
  }

  def apply(input: JValue): TableRow = {
    apply(input.asInstanceOf[JObject])
  }

  private def convert(value: JValue): AnyRef = {
    def fail() = throw new ArquiveiException("Failed To Convert From JObject to TableRow")

    value match {
      case JNothing => fail()
      case JNull => fail()
      case JString(x) => new java.lang.String(x)
      case JDouble(x) => new java.lang.Double(x)
      case JDecimal(x) => new java.lang.Double(x.doubleValue())
      case JLong(x) => new java.lang.Long(x)
      case JInt(x) => new java.lang.Long(x.longValue())
      case JBool(x) => new java.lang.Boolean(x)
      case JObject(x) => {
        val row = new TableRow()
        for ((k, v) <- x) {
          row.set(k, convert(v))
        }
        row
      }
      case JArray(x) => new java.util.ArrayList[AnyRef] (x.map(JsonToTablerow.convert).asJava)
      case JSet(x) => fail() // not supported
    }
  }
}

```

Na posse do ```TableRow```, já podemos inserir em uma tabela do BigQuery. Entretanto, o nome da tabela será dado dinamicamente, então é necessário um método para inferir o nome da tabela de destino a partir do conteúdo do evento, esse método será enviado para a lib do Apache Beam responsável por fazer a inserção.

Implementamos essa função baseando-se no nosso envelope de eventos:

```scala

  def getTableSpec(input: TableRow): String = {
    val illegalChars = new Regex("""[^a-zA-Z0-9\_]+""")
    val eventNameRaw = input.getOrDefault("Type", fallbackTableId).toString
    val eventName = illegalChars.replaceAllIn(eventNameRaw, "_").toLowerCase
    val sourceNameRaw = input.getOrDefault("Source", "fallbackSource").toString
    val sourceName = illegalChars.replaceAllIn(sourceNameRaw, "_").toLowerCase
    if (eventName == fallbackTableId) {
      fallbackTableSpec
    } else {
      TableFullName(projectId, datasetId, s"${sourceName}_$eventName")
    }
  }

// Com:
// fallbackTableId = Tabela de fallback
// illegalChars =  Regex que irá assegurar que o nome final da tabela do BigQuery só contém caracteres válidos
// projectId = projeto do GCP
// datasetId = dataset do BigQuery
// fallbackTableSpec = caminho completo para a tabela de fallback: projectId:datasetId.fallbackTableId
// TableFullName = método que obtém caminho completo de tabela no BigQuery: projectId:datasetId.tableId
```
Para inserção no BigQuery, utilizamos uma classe chamada [DynamicDestinations](https://beam.apache.org/releases/javadoc/2.9.0/org/apache/beam/sdk/io/gcp/bigquery/DynamicDestinations.html) do ApacheBeam, que nos permite utilizar uma função que interpreta dinamicamente os dados de um evento que chega a fim de obter o nome da tabela (método ```getTableSpec```) de destino no BigQuery.

Abaixo é apresentada a implementação da ```PTransform``` (uma das principais abstrações do Apache Beam) da primeira tentativa de inserção no BigQuery:

```scala
  class WriteToBigQuery(getTableSpec: TableRow => String) extends PTransform[PCollection[TableRow], PCollection[TableRow]] {

    override def expand(input: PCollection[TableRow]): PCollection[TableRow] = {
      input.apply(s"Write To BigQuery",
        BigQueryIO.writeTableRows().to(new DynamicDestinations[TableRow, TableRow] {

          override def getDestination(inputElement: ValueInSingleWindow[TableRow]): TableRow = {
            inputElement.getValue
          }

          override def getTable(inputElement: TableRow): TableDestination = {
            val tableReference = getTableSpec(inputElement)
            new TableDestination(tableReference, null, new TimePartitioning().setType("DAY"))
          }

          override def getSchema(inputElement: TableRow): TableSchema = {
            val inputJson = new Gson().toJson(inputElement) //Gson -> Lib do Google para converter TableRow em String JSON            
            JValueToTableSchema(inputJson)
          }

        })
          .withFailedInsertRetryPolicy(InsertRetryPolicy.retryTransientErrors())
          .withWriteDisposition(WriteDisposition.WRITE_APPEND)
      ).getFailedInserts
    }
  }
```


Apesar de conter poucas linhas, o código acima carrega uma altíssima complexidade.

Primeiramente, o método ```getDestination``` é executado. Esse método pode usar uma função que seleciona um evento dentre muitos a partir de um [janelamento/windowing](https://beam.apache.org/documentation/programming-guide/#windowing), porém, na nossa implementação não estamos utilizando nenhum janelamento personalizado, apenas a janela global.

Então, o evento escolhido é passado para a função ```getTable```, que pode receber qualquer lógica para obtenção do nome da tabela final. A partir da obtenção do destino do evento, o método tentará inserir o dado no BigQuery caso a tabela de destino já exista. Caso a tabela não exista, o método getSchema será executado, para obter um Schema da tabela de destino a partir da implementação dessa função.

Nossa implementação utiliza a função ```JValueToTableSchema```, que obtém um Schema de tabela do BigQuery a partir de um ```JValue```.

Nesse momento temos que fazer escolhas padronizadas para tipos de dados de destino do BigQuery a partir dos tipos inseridos no Json:

```scala
  def JValueToTableSchema(inputRow: JValue): TableSchema = {

    val jValueMap = inputRow.noNulls.extract[Map[String, JValue]]
    new TableSchema().setFields(jValueMap.map(
      {
        case (fieldName, fieldValue) =>
          convert(fieldName, fieldValue, "NULLABLE")
      }).toList.asJava
    )
  }

  def convert(fieldName: String, fieldValue: JValue, mode: String): TableFieldSchema = {

    def fieldSchema(typeStr: String, modeStr: String = mode, fieldNameStr: String = fieldName): TableFieldSchema =
      new TableFieldSchema().setName(fieldNameStr).setMode(modeStr).setType(typeStr)

    fieldValue match {
      case JBool(_) => fieldSchema("BOOLEAN")
      case JInt(_) => fieldSchema("NUMERIC")
      case JLong(_) => fieldSchema("NUMERIC")
      case JDecimal(_) => fieldSchema("NUMERIC")
      case JDouble(_) => fieldSchema("NUMERIC")
      case JString(inputString) => stringToTableSchema(fieldName, inputString)
      case JObject(listOfJFields) => jObjectToTableSchema(fieldName, listOfJFields, mode)
      case JArray(listOfJValue) => convertJArray(fieldName, listOfJValue)
      case _ => throw new ArquiveiException("InvalidField")
    }
  }

  def jObjectToTableSchema(fieldName: String, listOfJFields: List[JField], mode: String): TableFieldSchema = {
    new TableFieldSchema().setName(fieldName).setMode(mode).setType("RECORD").setFields(
      listOfJFields.map(
        {
          case (name, value) =>
            convert(name, value, "NULLABLE")
        }).asJava
    )
  }

  def stringToTableSchema(fieldName: String, inputString: String, mode: String = "NULLABLE"): TableFieldSchema = {
    def fieldSchema(typeStr: String, modeStr: String = mode, fieldNameStr: String = fieldName): TableFieldSchema =
      new TableFieldSchema().setName(fieldNameStr).setMode(modeStr).setType(typeStr)

      if (isEventTimestamp(inputString)) {
      fieldSchema("TIMESTAMP")
      } else if (isEventDate(inputString)) {
        fieldSchema("DATE")
      } else {
        fieldSchema("STRING")
      }
  }

  def convertJArray(fieldName: String, listOfJValue: List[JValue]): TableFieldSchema = {

    def fieldSchema(typeStr: String, modeStr: String, fieldNameStr: String = fieldName): TableFieldSchema =
      new TableFieldSchema().setName(fieldNameStr).setMode(modeStr).setType(typeStr)

    listOfJValue match {
      case List(_: JInt, _*) => fieldSchema("NUMERIC", modeStr = "REPEATED")
      case List(_: JBool, _*) => fieldSchema("BOOLEAN", modeStr = "REPEATED")
      case List(_: JLong, _*) => fieldSchema("NUMERIC", modeStr = "REPEATED")
      case List(_: JDecimal, _*) => fieldSchema("NUMERIC", modeStr = "REPEATED")
      case List(_: JDouble, _*) => fieldSchema("NUMERIC", modeStr = "REPEATED")
      case List(_: JString, _*) =>
        if (!listOfJValue.forall({ inputJString => isEventTimestamp(inputJString.extract[String]) })) {
          if (!listOfJValue.forall({ inputJString => isEventDate(inputJString.extract[String]) })) {
            fieldSchema("STRING", modeStr = "REPEATED")
          } else {
            fieldSchema("DATE", modeStr = "REPEATED")
          }
        } else {
          fieldSchema("TIMESTAMP", modeStr = "REPEATED")
        }
      case List(_: JArray, _*) => throw new ArquiveiException("BigQuery will not Accept Array of Array")
      case List(_: JObject, _*) =>
        val tableSchema = listOfJValue.foldLeft(new TableSchema())(
          {
            (accumulatedTableSchema: TableSchema, iteratedJField: JValue) =>
              val iteratedTableSchema = iteratedJField match {
                case JObject(listOfJFields) => new TableSchema().setFields(
                  listOfJFields.map(
                    {
                      case (name, value) =>
                        convert(name, value, "NULLABLE")
                    }).asJava
                )
                case _ => throw new ArquiveiException("All elements in this array must be of the same type (JObject)")
              }
              MergeTableSchema.createNewTableSchema(accumulatedTableSchema, iteratedTableSchema) // método criado para dar merge em dois Schemas
          })
        new TableFieldSchema().setName(fieldName).setMode("REPEATED").setType("RECORD").setFields(tableSchema.getFields)
    }
  }

```

Essa função é bem complexa, porém, somente com uma lógica de obtenção de Schema a partir de um evento que conseguimos atingir o objetivo proposto.

Por fim, a parte final da implementação da nossa ```PTransform``` possibilita tratarmos os casos inesperados:

```scala
          .withFailedInsertRetryPolicy(InsertRetryPolicy.retryTransientErrors())
          .withWriteDisposition(WriteDisposition.WRITE_APPEND)
      ).getFailedInserts
```
A política de Retry garante que serão feitas várias tentativas de inserção nas tabelas já existentes com Schema já definido, com retentativa para erros transientes (como indisponibilidade do BigQuery, etc). Caso várias tentativas sejam feitas sem sucesso, o ".getFailedInserts" nos devolve um conjunto de dados com as falhas de inserção.

Essas falhas ocorrem em duas situações:

- O JSON digievoluiu e ganhou mais campos \o\ /o/ \o/
- O JSON falhou na inserção por não respeitar nossas condições (exemplo: uma string tentou ser inserida num campo numérico) :( :(

Para as duas situações temos solução.

Agora, forçamos o não paralelismo utilizando um recurso do Apache Beam que é o [Stateful Processing](https://beam.apache.org/blog/2017/02/13/stateful-processing.html) e para cada evento que falha, nós:

- Obtemos o Schema do evento a partir do método ```JValueToTableSchema```
- Obtemos o Schema da tabela de destino a partir da API do BigQuery
- Comparamos os dois Schemas e fazemos um merge dos dois, para respeitar o nosso JSON digievoluído e também suas versões mais antigas
- Realizamos uma mutação do Schema da tabela de Destino

A mutação da tabela de destino pode falhar. Isso indica que nossas condições não foram respeitadas, pois um campo teria seu tipo alterado por um update de Schema (o que o BigQuery não permite).

Se a mutação de tabela falhar, entra a nossa maravilhosa "fallback_table". Nós pegamos o evento que falhou e colocamos ele em um envelope menos sexy, porém utilizável para os casos de falha:

```scala
  def parseToFallback(inputRow: TableRow): TableRow = {

    val formats = org.json4s.DefaultFormats
    inputRow.remove("ProcessingTime")

    val jsonStr = new Gson().toJson(inputRow)

    val payloadObj = Try(parse(jsonStr)) match {
      case Success(obj) => obj
      case Failure(_) =>
        LOG.warn(s"Problem parsing message! It may not be a valid message: $jsonStr")
        JObject()
    }

    val processingTime = Instant.now()
    val processingTimeStr = arquiveiDateFormat.format(processingTime)

    val output = new TableRow()
      .set("Id", (payloadObj \ "Id").extractOrElse[String]("WithoutId"))
      .set("Source", (payloadObj \ "Source").extractOrElse[String]("WithoutSource"))
      .set("Type", (payloadObj \ "Type").extractOrElse[String]("WithoutType"))
      .set("MessagePayload", jsonStr)
      .set("CreatedAt", (payloadObj \ "CreatedAt").extractOrElse[String]("WithoutCreatedAt"))
      .set("ProcessingTime", processingTimeStr)
    output
  }
```

Dessa forma, o conteúdo do evento fica dentro do MessagePayload. De milhões e milhões de eventos que processamos, apenas algumas centenas acabaram caíndo nessa tabela \o/ e posteriomente adaptamos os schemas e reportamos os bugs para os produtores.

No caso da mutação de tabela ter sucesso, então tentamos inserir novamente no BigQuery usando a mesma PTransform que apresentamos (WriteToBigQuery).

Se por algum motivo existir alguma falha, realizamos o mesmo procedimento e enviamos o dado para a tabela de fallback.

Dessa forma então finalizamos a apresentação das abordagens de implementação do pipeline. 

Ficamos à disposição de todos :)
