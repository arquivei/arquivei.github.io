---
layout: post
title: Consumindo uma lista de forma eficiente utilizando Go
date: 2018-12-07
categories: blog
img: golang/gopher_couch.png
tags: [Golang, Go, Parallel, Routines, Producer Consumer, High Performance]
author: victor_nunes
brief: Como consumir uma lista em Go do jeito mais rápido possível considerando diferentes cenários?
---
<style>
  .center {
    display: block;
    margin: 0 auto;
}
</style>

Uma das promessas da linguagem de programação Go é a utilização de suas funções nativas para construir uma arquitetura simples e rápida. Este artigo mostra algoritmos com essa característica que pode ser utilizada para processar rapidamente uma lista de objetos, utilizando três cenários distintos que vemos no cotidiano.

# Um pouco sobre Go

Go (ou GoLang) é uma linguagem de programação criada pela Google em novembro de 2009. Segundo [sua própria documentação](https://golang.org/doc/), a linguagem é um projeto _open source_ que tem como objetivo fazer os programadores serem mais produtivos. Go também é definido como uma linguagem expressiva, concisa, limpa e eficiente. Seu mecanismo de concorrência facilita escrever programas que aproveitam o máximo de máquinas multicore, enquanto o seu sistema de tipos permite a construção de programas flexíveis e modulares. Além disso, Go é uma linguagem compilada, é _memory safety_, possui um _garbage collector_ e é fortemente tipada.

![gopher-dance]({{site.baseurl}}/assets/img/posts/golang/dance.gif){:class="center"}

## Paralelismo em Go

Gorotinas (ou simplemente **rotinas**) são _lightweight threads_ gerenciadas pelo _runtime_ do Go. Essas rotinas são executadas no mesmo espaço de endereçamento, então, elas acessam memória compartilhada e devem ser sincronizadas. Para criar uma rotina, basta adicionar a palavra **go** antes da invocação da função. Exemplo de criação de rotinas:

```go
// cria uma rotina para executar a função doSomething()
go doSomething() 

// cria uma rotina para executar uma função anônima que possui uma lógica interna
go func() { 
  fmt.Println("hello")
  doAnotherThing()
}()
```

As rotinas podem ser sincronizadas utilizando barreiras. Umas das formas de se utilizar barreiras em Go é por meio da estrutra `sync.WaitGroup` e possui os seguintes métodos:

```go
wg := sync.WaitGroup{} // cria barreira
wg.Add(N) // adiciona N elementos na barreira
wg.Done() // subtrai 1 elemento da barreira
wg.Wait() // bloqueia rotina até barreira ficar liberada
```

## Tipos de dados em Go

Go possui vários tipos de dados nativos. Os mesmos podem ser estudados e praticados no _tour_ oficial da linguagem ([inglês](https://tour.golang.org/moretypes/1), [português](https://go-tour-br.appspot.com/moretypes/1)). Para o entendimento deste artigo, será explicado brevemente dois tipos: **listas** e **canais**.

### Listas

Listas em Go podem ser estruturadas de dois modos. O primeiro, chamado _Array_, é representado como `[n]T`, sendo **n** a quantidade de valores do tipo **T** que a lista pode possuir. O segundo, chamado _Slice_, é representado como `[]T` e é uma referência para um _Array_. Essa referência pode ser explícita ou implícita (o _runtime_ alocará automaricamente um _Array_). Exemplos de lista: 

```go
// i é um slice de inteiros
var i []int 

// s é um array de 10 elementos de strings
var s [10]string 
```

Neste artigo iremos trabalhar com listas do tipo _Slice_. Para saber mais sobre lista, como inicializar e iterar sobre elas, o _tour_ oficial possui um módulo para isso ([inglês](https://tour.golang.org/moretypes/6), [português](https://go-tour-br.appspot.com/moretypes/6)).

### Canais

Canais são estuturas de dados utilizadas para duas ou mais rotinas se comunicarem. Os mesmos são bloqueantes na leitura e na escrita, ou seja, quando uma rotina tenta produzir um elemento em um canal cheio, ela é bloqueada até alguma outra rotina consumir pelo menos um item do canal, e quando uma rotina tenta consumir de um canal vazio, ela é bloqueada até alguma outra rotina escrever no canal.

Os valores de um canal podem ser produzidos e consumidos a partir do operador `<-`, onde a informação é transmitida seguindo sempre esse sentido da seta (sempre da direita para a esquerda).

```go
// inserindo o valor "1" no canal "ch"
ch <- 1 

// lendo um valor do canal "ch" e atribuindo à variável "i"
i := <- ch 
```

Quando um produtor parar de inserir elementos em um canal, ele pode fechar o mesmo para indicar para o consumidor que as informações acabaram.

```go
// fechando o canal "ch"
close(ch) 
```

O consumidor pode consumir de um canal de várias formas. Uma delas foi apresentada acima, através do operador `<-`. Outra forma é iterar no canal. Essa iteração é realizada até o canal ser fechado, bloqueando a rotina enquanto não tem nada para ser lido.

```go
// iterando sobre o canal "ch" até que o mesmo seja fechado
for i := range ch {
    fmt.Println(i)
}
```

# Problema de consumir de uma lista

A pergunta que este artigo tentará responder é como consumir uma fila de forma eficiente considerando a latência e a memória utilizada. Para fins de exemplo, será considerado que deseja-se consumir uma fila de inteiros e inserir a resposta em outra fila de inteiros. O algoritmo que deseja-se deixar mais eficiente pode ser representado da seguinte forma:

```go
// consumidor de fila sequencial
func process(in []int) (out []int) {
    out = make([]int, len(in))
    for i, input := range in {
        out[i] = inputProcess(input)
    }
    return
}

// inputProcess processa a entrada, gerando uma saída
func inputProcess(in int) (out int) {
    return
}
```

# Cenários considerados

A solução para este problema, como já apresentado, é propor implementações em Go para consumir a lista do jeito mais rápido possível. Para que a frase _**"do jeito mais rápido possível"**_ seja verdadeira, é necessário criar uma solução diferente para cada cenário em que ocorre o problema. Dessa forma, este artigo irá tratar 3 cenários diferentes. Para todos esses exemplos, será utilizado **N** como o número máximo de rotinas que poderão ser executadas ao mesmo tempo. Esse número é definido pelos desenvolvedores para que não ocorra muito consumo de memória.

Os dois primeiros cenários consideram o tamanho da entrada. O primeiro consiste em uma lista de **entrada pequena** (menor que N). O segundo consiste em uma lista de **entrada grande** (maior que N). Para esses dois exemplos, será considerado que o tamanho da saída é igual o tamanho da entrada.

O terceiro cenário consiste em uma **saída estruturada**. Essa saída pode conter informações de **ordem da entrada**, **erro do processamento da entrada** e outros **metadados** desejados. Nesse caso, a saída não necessariamente terá o mesmo tamanho que a entrada.

# Soluções

## 1) Entrada pequena

Será considerarado que não há erros de processamento da entrada.

```go
func process(in []int) (out []int) {
    // barreira
    wg := sync.WaitGroup{}
    wg.Add(len(in))

    // gera uma rotina para cada elemento da entrada
    out = make([]int, len(in))
    for i, input := range in { 
        go func(i, input int) {
            defer wg.Done()   
            out[i] = inputProcess(input)
        }(i, input)
    }

    // espera barreira
    wg.Wait()

    return
}

// inputProcess processa a entrada, gerando uma saída
func inputProcess(in int) (out int) {
    return
}
```

## 2) Entrada grande

Será considerarado que a saída pode ser desordenada e que não há erros de processamento da entrada. Esta solução trabalha com _workers_, onde cada um executa uma rotina diferente e todas as rotinas consomem a entrada. Para isso não causar uma condição de disputa, as rotinas consomem de um canal auxiliar `inChan`, onde as entradas serão processadas e inseridas no mesmo. É necessário utilizar um canal de saída também, `outChan`, pois a operação _append_ em uma lista não é _memory safety_.

```go
func producerConsumer(in []int) (out []int) {
    // limitando número de workers em N
    const maxWorkers = N
    nWorkers := len(in)
    if nWorkers > maxWorkers {
        nWorkers = maxWorkers
    }

    // inicializando canais auxiliares para processar a entrada e a saída
    inChan := make(chan int)
    outChan := make(chan int)

    // barreira
    wg := sync.WaitGroup{}
    wg.Add(nWorkers)

    // consumidor do canal auxiliar de entrada
    // produtor do canal auxiliar de saída
    // gera uma rotina para cada worker
    for i := 0; i < nWorkers; i++ {
        go func() {
            defer wg.Done()
            for input := range inChan {
                outChan <- inputProcess(input)
            }
        }()
    }

    // consumidor da entrada
    // produtor do canal auxiliar de entrada
    // espera barreira para fechar canal de saída
    go func() {
        for _, i := range in {
            inChan <- i
        }
        close(inChan)
        wg.Wait()
        close(outChan)
    }()

    // consumidor do canal auxiliar de saída
    // produtor da saída
    for o := range outChan {
        out = append(out, o)
    }

    return
}

// inputProcess processa a entrada, gerando uma saída
func inputProcess(in int) (out int) {
    return
}
```

## 3) Saída estruturada

Será considerarado uma entrada grande. Além disso, será considerado que é desejado que a saída possua a mesma **ordem** da lista de entrada e que deve ser considerado o **erro** de processamento da entrada.

```go
func producerConsumer(in []int) (out []int) {
    // limitando número de workers em N
    const maxWorkers = N
    nWorkers := len(in)
    if nWorkers > maxWorkers {
        nWorkers = maxWorkers
    }

    // estrutura auxiliar
    type info struct {
        value int
        position int
        err error
    }

    // inicializando canais auxiliares para processar a entrada e a saída
    inChan := make(chan info)
    outChan := make(chan info)

    // barreira
    wg := sync.WaitGroup{}
    wg.Add(nWorkers)

    // consumidor do canal auxiliar de entrada
    // produtor do canal auxiliar de saída
    // gera uma rotina para cada worker
    for i := 0; i < nWorkers; i++ {
        go func() {
            defer wg.Done()
            for input := range inChan {
                o, err := inputProcess(input.value)
                outChan <- info {
                    value: o,
                    position: input.position,
                    err: err,
                }
            }
        }()
    }

    // consumidor da entrada
    // produtor do canal auxiliar de entrada
    // espera barreira para fechar canal de saída
    go func() {
        for i, input := range in {
            inChan <- info{
                value: input,
                position: i,
            }
        }
        close(inChan)
        wg.Wait()
        close(outChan)
    }()

    // consumidor do canal auxiliar de saída
    // produtor da saída
    out = make([]int, len(in))
    for o := range outChan {
        if o.err != nil {
            // fazer alguma coisa com o erro,
            // podendo loggar ele ou retornar
        }
        out[o.position] = o.value
    }
    
    return
}

// inputProcess processa a entrada, gerando uma saída
func inputProcess(in int) (out int, err error) {
    return
}
```

## Outras soluções

Para considerar erros de processamento na Solução 1, deve-se utilizar um canal de saída assim como feito na Solução 2. Isso deve ser feito pois a solução apresentada considera que cada elemento da lista de entrada terá seu correspondente na lista de saída. Quando erros começam a ser considerados, essa afirmação deixa de ser verdade, fazendo com que seja necessário inserir na lista de saída com um _append_ ao invés de utilizar o índice. Como o _append_ em uma lista não é _memory safety_, deve ser utilizado o canal de saída.

É bom lembrar que a entrada e a saída podem estar estruturadas de outra forma. Há situações em que ao invés de uma lista se tem um canal com muitos dados a serem processados. Neste caso, as soluções seriam as mesmas que as apresentadas neste artigo, porém, sem a construção do `inChan`. Analogamente, pode ser que a resposta fique em um canal, podendo ignorar o **produtor de saída**. Neste último caso, no lugar do **produtor de saída**, a rotina teria que ficar bloqueada com o `wg.Wait()`.


# Resultados

Para analisar os resultados, as soluções serão comparadas com a versão sequencial, apresentada na seção **Problema de consumir de uma lista**. Além disso, será comparado somente as duas primeiras soluções, pois as mesmas serão feitas com base no tamanho da lista. Em todos os testes, toda função `inputProcess` tem uma espera de 1ms e uma alocação grande de memória. Por fim, o número máximo de rotinas executadas ao mesmo tempo, **N**, foi definido como 50.

![solucao-entrada-pequena]({{site.baseurl}}/assets/img/posts/golang/graph_solution1.png){:class="center"}
![solucao-entrada-grande]({{site.baseurl}}/assets/img/posts/golang/graph_solution2.png){:class="center"}
![comparação-entre-solucoes]({{site.baseurl}}/assets/img/posts/golang/graph_solution1n2.png){:class="center"}

Os gráficos indicam que as soluções propostas foram muito mais eficiêntes que a solução sequencial. Além disso, comparando as duas soluções, é possível notar duas características: para entrada **menor que N** (sendo N=50), a Solução 1 é mais rápida e consome a mesma quantidade de memória que a Solução 2; porém, para entrada **maior que N**, a Solução 2, apesar de ser mais lenta, tem um consumo constante de memória, enquanto a Solução 1 tem um consumo crescente.

# Conclusão

Utilizar Go para construir um algoritmo de alto desempenho se mostrou uma alternativa muito boa. Apesar de ser uma linguagem nova no mercado, vale a pena estudá-la por sua simplicidade em lidar com arquiteturas que lidam com muito paralelismo. Além disso, vale ressaltar que tratar o problema apresentado de forma específica aumentou a eficiência do programa. Obrigado pela leitura e mãos à obra!

![gopher-gym]({{site.baseurl}}/assets/img/posts/golang/gym.gif){:class="center"}
