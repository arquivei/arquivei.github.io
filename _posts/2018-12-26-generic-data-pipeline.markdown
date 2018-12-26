---
layout: post
title: Criando e evoluindo tabelas no BigQuery de maneira genérica a partir de UM pipeline de dados em Streaming
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
utilizamos um wrapper do Spotify (SCIO) em Scala para o Apache Beam, um modelo de programação voltado para processamento de dados em streaming e batch, que executa no Google Dataflow e em outros runners de processamento de dados, como o Apache Spark e Flink.


# Contexto

Na Arquivei, possuímos centenas de eventos em JSON chegando com Schemas diferentes via streaming. Temos a necessidade de executar queries sobre os dados desses eventos e para isso utilizamos o Google BigQuery como Data Warehouse.<br>
O fato de possuirmos centenas de eventos diferentes trouxe um problema de produtividade e escalabilidade no time de dados, porque para cada novo tipo de evento, ou para cada novo campo em um evento, existia a necessidade de fazer uma alteração e novo deploy nos pipelines de dados existentes.<br>

## Abordagem não escalável
A necessidade de realizar um novo deploy para cada alteração ou novo evento se dava pelo fato das definições de schema e da tabela de destino do evento estarem nos códigos dos pipelines. Os pipelines faziam a leitura do JSON, checava-se os campos Source e Type e os eventos que não deveriam ser processados eram filtrados. Os eventos selecionados eram parseados conforme a definição de seu Schema. A tabela de destino no BigQuery então era criada ou atualizada conforme o novo schema (o que causava a interrupção do pipeline para Deploy).<br><br>
Nesse momento estávamos utilizando o [Apache Beam](https://beam.apache.org/) em Java.<br><br>
Abaixo está um exemplo de código que realiza o parse de um evento de Login.
No modelo de programação do Apache Beam, deve-se implementar a classe DoFn, que será distribuída para processamento paralelo.<br>
De maneira simplista, uma função idempotente (processElement) realiza uma leitura (context.element()) e uma escrita (context.output) do dado processado.
Nesse caso o tipo de dado utilizado é o TableRow, que representa uma linha de dados no BigQuery.
<br><br>

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

Temos uma padronização do formato de eventos que já facilitava esse trabalho. Os eventos devem vir em um JSON válido, seguindo esse envelope:

```json

{
    "SchemaVersion": 1,
    "ID": "a-random-id",
    "Source": "event-source",
    "Type": "event-type",
    "CreatedAt": "2017-01-01T23:24:25.123456789-03:00",
    "DataVersion": 1,
    "Data": {}
}
```

O envelope garante que os dados relacionados ao evento ficam dentro do campo Data e que algumas informações importantes que todos os eventos possuem já se encontram disponíveis nos campos SchemaVersion, ID, Source, Type, CreatedAt e DataVersion.

Um exemplo de evento de login a ser processado pelo código apresentado acima seria:

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



## E agora?

Começamos a buscar várias alternativas para resolver esse problema de escalabilidade e de produtividade, até que encontramos [um artigo no blog do Google](https://cloud.google.com/blog/products/gcp/how-to-handle-mutating-json-schemas-in-a-streaming-pipeline-with-square-enix/) descrevendo um problema praticamente idêntico ao nosso, apresentando como solução as novas bibliotecas do Apache Beam para BigQuery, porém sem entrar em muitos detalhes de como fazer, nem apresentar exemplos de código mais detalhados. Utilizando a ideia apresentada nesse artigo, decidimos criar esse post para apresentar a nossa implementação da resolução desse problema.

# O "pipeline genérico"

Carinhosamente apelidado pelo time de Engenharia de Dados de "Pipeline genérico", desenvolvemos um pipeline para substituir quilos de código por apenas apenas alguns métodos mágicos que iriam abstrair o conceito de parsear eventos e inseri-los no BigQuery.

A ideia geral do pipeline segue esse fluxo apresentado no artigo do Google:

![genericpipe-flow]({{site.baseurl}}/assets/img/posts/genericpipeline/mutating-json-2a2es.max-900x900.PNG){:class="center"}
<center>Retirado do Blog do Google Cloud</center>

## As ferramentas

Para melhorar a velocidade na entrega dos códigos e legibilidade <strike>(Java, eca)</strike> decidimos migrar os pipelines de Java para a linguagem [Scala](https://www.scala-lang.org/), que é interoperável com Java e combina orientação a objetos com programação funcional.

Como estávamos utilizando as bibliotecas puras de Apache Beam para Java, passamos a utilizar um wrapper do Apache Beam para Scala, que é [um projeto open-source do Spotify chamado Scio](https://github.com/spotify/scio).

Como infraestrutura de mensageria, utilizamos o Apache Kafka, porém qualquer serviço de mensageria em Streaming que tenha uma source facilmente instanciável no Apache Beam, como o Google Pubsub, pode ser utilizado sem dores de cabeça.

Como Data Warehouse, utilizamos o [BigQuery](https://cloud.google.com/bigquery/), altamente escalável, barato, integrável com uma série de ferramentas de Business Intelligence e completamente gerenciado pelo Google. Claro que tudo depende do caso de uso, mas para casos de uso parecidos com os nossos já vimos empresas perdendo MUITO tempo partindo para soluções Hadoop. Esse artigo limita-se no BigQuery como Data Warehouse, pelo fato da utilização de suas bibliotecas dinâmicas do Apache Beam. 

Por fim, como runner dos pipelines do Apache Beam, utilizamos o [Dataflow](https://cloud.google.com/dataflow/?hl=pt-br), também gerenciado pelo Google. Aqui, as alternativas são diversas e são apresentadas na [página do Apache Beam](https://beam.apache.org/documentation/runners/capability-matrix/). Atualmente, os runners de pipelines do Apache Beam são:
Google Cloud Dataflow, Apache Flink, Apache Spark, Apache Apex, Apache Gearpump, Apache Hadoop MapReduce, JStorm, IBM Streams, Apache Samza e o Direct Runner (sem um runner específico).

## Condições

Precisamos fazer alguns assumptions para o funcionamento com perfeição do nosso "Pipeline genérico". Estamos utilizando o JSON como protocolo de transmissão de dados. Esses assumptions poderiam ser garantidos pela mudança para um formato como o [AVRO](https://avro.apache.org/), que garantiria um problema no produtor de eventos caso ele não esteja produzindo no padrão esperado. Isso pode ser alcançado com a implantação do [Schema Registry](https://github.com/confluentinc/schema-registry) para o Apache Kafka e esse é o próximo passo que provavelmente faremos nos esforços de escalabilidade e padronização.

Atualmente os assumptions são:
- O evento é um JSON válido
- O evento segue o nosso envelope padrão (como apresentado no início do post)
- Um evento nunca terá um tipo de dado alterado em algum campo existente (exemplo de problema: {"AccountId": 1} -> {"AccountId": '1'})

Não é o fim do mundo caso um dos assumptions seja quebrado (como já foi) pois temos algumas garantias que vou explicar posteriormente.

## Show me the code

Primeiramente, estamos utilizando o [JSON4s](http://json4s.org/), uma representação AST (Abstract Syntax Tree) para o JSON em Scala. O uso do Json4s não é obrigatório, porém os códigos que serão apresentados utilizarão essa lib.

Com os eventos chegando como String, nós realizamos a conversão de String para JValue

Esse método e vários outros, utilizarão do [Pattern Matching de Scala](https://docs.scala-lang.org/tour/pattern-matching.html), elemento comum em linguagens funcionais:

```scala
  def parseMessage(msg: String): JValue = {
    implicit val formats = org.json4s.DefaultFormats
    Try(parse(msg)) match {
      case Success(obj) => obj
      case Failure(_) =>
        LOG.error(s"Problem parsing message! It may not be a valid message: $msg")
        JNull
    }
  }
```
Nesse momento, um erro é adicionado no LOG caso a string não siga o padrão JSON. Como utilizamos o Google Dataflow, criamos um aviso via [Stackdriver](https://cloud.google.com/stackdriver/), que nos envia um e-mail caso um erro chegue ao LOG.

Com o JValue, é o momento de convertê-lo para TableRow, que é a representação do Apache Beam para uma linha numa tabela do BigQuery, então nós precisamos de um método para realizar a conversão entre JValue (o objeto JSON para o JSON4s) e TableRow.

```scala
object JsonToTablerow {
  def apply(input: JObject): TableRow = {
    implicit val format = org.json4s.DefaultFormats
    val sanitizedInput = input.noNulls remove {
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

É necessário um método para inferir o nome da tabela de destino a partir do conteúdo do evento, esse método será enviado para a lib do Apache Beam responsável por fazer a inserção.

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
Para inserção no BigQuery, utilizamos uma classe chamada [DynamicDestinations](https://beam.apache.org/releases/javadoc/2.9.0/org/apache/beam/sdk/io/gcp/bigquery/DynamicDestinations.html) do ApacheBeam, que nos permite utilizar uma função que interpreta dinamicamente os dados de um evento que chega a fim de obter o nome da tabela (método getTableSpec) de destino no BigQuery.

Abaixo é apresentada a implementação da PTransform da primeira tentativa de inserção no BigQuery, uma das principais abstrações do Apache Beam:

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

Primeiramente, o método getDestination é executado. Esse método pode usar uma função que seleciona um evento dentre muitos a partir de um [janelamento/windowing](https://beam.apache.org/documentation/programming-guide/#windowing), porém, na nossa implementação não estamos utilizando nenhum janelamento personalizado, apenas a janela global.

Então, o evento escolhido é passado para a função getTable, que pode receber qualquer lógica para obtenção do nome da tabela final. A partir da obtenção do destino do evento, o método tentará inserir o dado no BigQuery caso a tabela de destino já exista. Caso a tabela não exista, o método getSchema será executado, para obter um Schema da tabela de destino a partir da implementação dessa função.

Nossa implementação utiliza a função JValueToTableSchema, que obtém um Schema de tabela do BigQuery a partir de um JValue.

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

Por fim, a parte final da implementação da nossa PTransform possibilita tratarmos os casos inesperados:

```scala
          .withFailedInsertRetryPolicy(InsertRetryPolicy.retryTransientErrors())
          .withWriteDisposition(WriteDisposition.WRITE_APPEND)
      ).getFailedInserts
```
A política de Retry garante que serão feitas várias tentativas de inserção nas tabelas já existentes com Schema já definido, com retentativa para erros transientes (como indisponibilidade do BigQuery, etc). Caso várias tentativas sejam feitas sem sucesso, o ".getFailedInserts" nos devolve um conjunto de dados com as falhas de inserção.

Essas falhas ocorrem em duas situações:

- O JSON digievoluiu e ganhou mais campos \o\ /o/ \o/
- O JSON falhou na inserção por não respeitar nosso assumption (exemplo: uma string tentou ser inserida num campo numérico) :( :(

Para as duas situações temos solução.

Agora, forçamos o não-paralelismo utilizando um recurso do Apache Beam que é o [Stateful Processing](https://beam.apache.org/blog/2017/02/13/stateful-processing.html) e para cada evento que falha, nós:

- Obtemos o Schema do evento a partir do método JValueToTableSchema
- Obtemos o Schema da tabela de destino a partir da API do BigQuery
- Comparamos os dois Schemas e fazemos um merge dos dois, para respeitar o nosso JSON digievoluído e também suas versões mais antigas
- Realizamos uma mutação do Schema da tabela de Destino

A mutação da tabela de destino pode falhar. Isso indica que nossos assumptions não foram respeitados, pois um campo teria seu tipo alterado por um update de Schema (o que o BigQuery não permite).

Se a mutação de tabela falhar, entra a nossa maravilhosa "fallback_table". Nós pegamos o evento que falhou e colocamos ele em um envelope menos sexy, porém utilizável:

```scala
  def parseToFallback(inputRow: TableRow): TableRow = {

    implicit val formats = org.json4s.DefaultFormats
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


# Conclusão

Implementamos com sucesso o pipeline e tivemos um ganho enorme em termos de produtividade. Não nos preocupamos mais quando novos eventos são produzidos e ganhamos agilidade para nos dedicar em tarefas que trazem maior valor.

O pipelines está rodando com sucesso no Google Cloud Dataflow e já processou centenas de milhões de eventos!!