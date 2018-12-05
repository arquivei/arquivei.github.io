---
layout: post
title: Consumindo uma lista de forma eficiente utilizando Go
date: 2018-12-05
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

Uma das promessas da linguagem de programação Go é a utilização de suas funções nativas para construir uma arquitetura simples e rápida. Este artigo mostra uma arquitetura com essa característica que pode ser utilizada para processar rapidamente uma lista de objetos, utilizando três cenários distintos que vemos no cotidiano.

# Um pouco sobre Go

Go (ou GoLang) é uma linguagem de programação criada pela Google em novembro de 2009. Segundo [sua própria documentação](https://golang.org/doc/), a linguagem é um projeto _open source_ que tem como objetivo fazer os programadores serem mais produtivos. Go também é definido como uma linguagem expressiva, concisa, limpa e eficiente. Seu mecanismo de concorrência facilita escrever programas que aproveitam o máximo de máquinas multicore, enquanto o seu sistema de tipos permite a construção de programas flexíveis e modulares. Além disso, Go é uma linguagem compilada, é _memory safety_, possui um _garbage collector_ e é fortemente tipada.

![gopher-dance]({{site.baseurl}}/assets/img/posts/golang/dance.gif){:class="center"}

## Paralelismo em Go

Gorotinas (ou simplemente **rotinas**) são _lightweight thread_ gerenciadas pelo _runtime_ do Go. Essas rotinas são executadas no mesmo espaço de endereçamento, então, elas acessam memória compartilhada e devem ser sincronizadas. Para criar uma rotina, basta adicionar a palavra **go** antes da invocação da função. Exemplo de criação de rotinas:

```go
// cria uma rotina para executar a função doSomething()
go doSomething() 

// cria uma rotina para executar uma função anônima que possui uma lógica interna
go func() { 
  fmt.Println("hello")
  doAnotherThing()
}()
```

As rotinas podem ser sincronizadas utilizando barreiras. Uma barreira pode ser representada por uma estrutra `sync.WaitGroup` e possui os seguintes métodos:

```go
wg := sync.WaitGroup{} // cria barreira
wg.Add(N) // adiciona N elementos na barreira
wg.Done() // subtrai 1 elemento da barreira
wg.Wait() // bloqueia rotina até barreira ficar liberada
```

## Tipos de dados em Go

Go possui vários tipos de dados nativos. Os mesmos podem ser estudados e praticados no [_tour_](https://tour.golang.org/moretypes/1) oficial da linguagem. Para o entendimento deste artigo por parte das pessoas que não conhecem a linguagem, será explicado brevemente dois tipos: **listas** e **canais**.

### Listas

Uma lista em Go é estruturada de dois modos. O primeiro, chamado _Array_, é estruturado como `[n]T`, sendo **n** a quantidade de valores do tipo **T** que a lista pode possuir. O segundo, chamado _Slice_, é estruturado como `[]T`, e o mesmo referência explicitamente ou não um _Array_. Exemplos de lista: 

```go
// i é um slice de inteiros
var i []int 

// s é um array de 10 elementos de strings
var s [10]string 
```

Neste artigo iremos trabalhar com listas do tipo _Slice_. Para saber mais sobre lista, como inicializar e iterar sobre elas, o [_tour_](https://tour.golang.org/moretypes/6) oficial possui um módulo para isso.

### Canais

Canais são estuturas de dados utilizadas para duas ou mais rotinas se comunicarem. Os mesmos são bloqueantes na leitura e escrita, ou seja, quando uma rotina tenta produzir o (n+1)º elemento num canal (sendo **n** a capacidade máxima do canal), ela é bloqueada até alguém consumir do canal, e quando uma rotina tenta consumir de um canal vazio, ela é bloqueada até alguém escrever no canal.

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

O consumidor pode consumir de um canal de várias formas. Uma delas foi apresentada acima, através do operador `<-`. Outra forma é iterar no canal através de um `for`. Essa iteração é realizada até o canal ser fechado, bloqueando a rotina enquanto não tem nada para ser lido.

```go
// iterando sobre o canal "ch" até que o mesmo seja fechado
for i := range ch {
    fmt.Println(i)
}
```

# Problema de consumir de uma lista

O problema que este artigo tenta solucionar é o consumo não eficiente de uma fila ao considerar uma arquitetura sequencial. Para fins de exemplo, será considerado que deseja-se consumir uma fila de inteiros e inserir a resposta em outra fila de inteiros. Ou seja, a assinatura da função que soluciona o problema será a seguinte:

```go
func process(in []int) (out []int)
```

# Cenários considerados

A solução para este problema, como já apresentado, é propor arquituturas em Go para consumir a lista do jeito mais rápido possível. Para que a frase _**"do jeito mais rápido possível"**_ seja verdadeira, é necessário criar uma solução diferente para cada cenário em que ocorre o problema. Dessa forma, este artigo irá tratar 3 cenários diferentes. Para todos esses exemplos, será utilizado **N** como o número máximo de rotinas que poderão ser executadas ao mesmo tempo. Esse número é definido pelos desenvolvedores para que não ocorra muito consumo de memória.

Os dois primeiros cenários consideram o tamanho da entrada. O primeiro consiste em uma lista de **entrada pequena** (menor que N). O segundo consiste em uma lista de **entrada grande** (maior que N). 

O terceiro cenário consiste em uma **saída estruturada**. Essa saída pode conter informações de **ordem da entrada**, **erro do processamento da entrada** e outros **metadados** desejados.

# Soluções

## 1) Entrada pequena

Será considerarado que a saída pode ser desordenada e que não há erros de processamento da entrada. É necessário utilizar um canal nesta solução pois a operação e `append` em uma lista não é _memory safety_.

```go
func process(in []int) (out []int) {
    // inicializando canal auxiliar para processar a saída
    outChan := make(chan int)

    // barreira
    wg := sync.WaitGroup{}
    wg.Add(len(in))

    // consumidor da entrada
    // gera uma rotina para cada elemento da entrada
    for _, input := range in { 
        go func(input int) {
            defer wg.Done()   
            outChan <- inputProcess(input)
        }(input)
    }

    // espera barreira para fechar canal de saída
    go func() {
        wg.Wait()
        close(outChan)
    }()

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

## 2) Entrada grande

Será considerarado que a saída pode ser desordenada e que não há erros de processamento da entrada. Esta solução trabalha com _workers_, onde cada um executa uma rotina diferente e todas as rotinas consomem a entrada. Para isso não causar uma condição de disputa, as rotinas consomem de um canal auxiliar `inChan`, onde as entradas serão processadas e inseridas no mesmo.

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

    // consumidor da entrada
    // gera uma rotina para cada worker
    for i := 0; i < nWorkers; i++ {
        go func() {
            defer wg.Done()
            for input := range inChan {
                outChan <- inputProcess(input)
            }
        }()
    }

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

    // consumidor da entrada
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

    // produtor da saída
    out = make([]int, len(in))
    counter := 0
    for o := range outChan {
        if o.err != nil {
            // fazer alguma coisa com o erro,
            // podendo loggar ele ou retornar
        }
        out[o.position] = o.value
        counter++
    }
    out = out[0:counter]
    
    return
}

// inputProcess processa a entrada, gerando uma saída
func inputProcess(in int) (out int, err error) {
    return
}
```

## Outras soluções

Para ordenar a saída na situação de entrada pequena, pode ser utilizado o índice da iteração durante o processamento da lista de entrada.

É bom lembrar a entrada e a saída podem estar estruturadas de outra forma. Há situações em que ao invés de uma lista se tem um canal com muitos dados a serem processados. Neste caso, as soluções seriam as mesmas que as apresentadas neste artigo, porém, sem a construção do `inChan`. Analogamente, pode ser que a resposta fique em um canal, podendo ignorar o **produtor de saída**. Neste último caso, no lugar do **produtor de saída**, a rotina teria que ficar bloqueada, esperando todo o processamento finalizar. A liberação dessa rotina seria feita no lugar do `close(outChan)`.


# Resultados

Para fins de comparação, as soluções serão comparadas com a versão sequencial. A mesma será arquitetada como um simples `for` na lista de entrada, uma chamada para a função `inputProcess`, e uma inserção na saída. Além disso, será comparado somente as duas primeiras soluções, pois as comparações serão feitas com base no tamanho da lista. Em todos os testes, toda função `inputProcess` tem uma espera de 1ms e uma alocação grande de memória. Por fim, o número máximo de rotinas executadas ao mesmo tempo, **N**, foi definido como 50.

![solucao-entrada-pequena]({{site.baseurl}}/assets/img/posts/golang/graph_solution1.png){:class="center"}
![solucao-entrada-grande]({{site.baseurl}}/assets/img/posts/golang/graph_solution2.png){:class="center"}
![comparação-entre-solucoes]({{site.baseurl}}/assets/img/posts/golang/graph_solution1n2.png){:class="center"}

Os gráficos indicam que as soluções propostas foram muito mais eficiêntes que a solução sequencial. Além disso, comparando as duas soluções, é possível notar duas características: para entrada **menor que N** (sendo N=50), a solução para entrada pequena é mais rápida e consome a mesma quantidade de memória que a outra solução; porém, para entrada **maior que N**, a solução para entrada grande, apesar de ser mais lenta, tem um consumo constante de memória, enquanto a outra solução tem um consumo crescente.

# Conclusão

Utilizar Go para construir uma arquitetura de alto desempenho se mostrou uma alternativa muito boa. Apesar de ser uma linguagem nova no mercado, vale a pena estudá-la por sua simplicidade em lidar com arquiteturas que lidam com muito paralelismo. Além disso, vale ressaltar que tratar o problema apresentado de forma específica aumentou a eficiência do programa. Obrigado pela leitura e mãos à obra!

![gopher-gym]({{site.baseurl}}/assets/img/posts/golang/gym.gif){:class="center"}