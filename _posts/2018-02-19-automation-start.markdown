---
layout: post
title: Automation - A origem
date: 2018-03-07
categories: blog
img: automation-start.jpg
tags: [Python, Automation, Selenium, Test]
author: leticia
brief: O principal objetivo da automação de testes funcionais é eliminar o teste manual de cenários antigos, estáveis e repetitivos, agilizando o processo de deploy de uma nova funcionalidade.
---

Em abril de 2016 foi identificado a necessidade de se criar uma equipe de qualidade na Arquivei. Desde o início já existia o desejo de que a área surgisse com testes automatizados, porém foi em agosto que começamos os estudos para automatizar nossos fluxos. Por meados de dezembro de 2016, a automação do *Smoke Test* dava início, visando reduzir o tempo gasto com testes manuais das principais funcionalidades que poderiam ser afetadas pelo surgimento de novas.

Como temos uma plataforma com alto grau de dependência entre funcionalidades, é preciso retestar fluxos "estáveis" para garantir a qualidade da entrega da nova funcionalidade, o que gera muito retrabalho se feito de forma manual.


![automation-start-timeline]({{site.baseurl}}/assets/img/posts/automation-start-timeline.png){:class="img-fluid"}

# O que usar para Automatizar ?

A nossa necessidade era de tecnologias que fossem simples, confiáveis, de fácil manutenção e que nos desse maior liberdade para trabalhar. Buscando o que o mercado estava usando concluímos que gherkin (cucumber), selenium, lettuce e python seriam boas escolhas.

**Gherkin**     - por ser uma linguagem natural utilizada para descrever cenários de teste e que é fácil de ser compreendida também por pessoas da área de negócio (como o time de produto), e que também já pode ser interpretada por *frameworks* de *Behavior Driven Development*.

**Selenium**    - por ser a  ferramenta mais utilizada e confiável do mercado para testar aplicações web de forma automatizada gratuitamente.

**Python**      - por ser uma linguagem de programação com sintaxe simple e clara e por também oferecer recursos disponíveis em linguagens mais complicadas como Java.

**Lettuce**     - por ser um framework para BDD (Behavior Driven Development) que permite executar testes automáticos em Python de forma simples.


##### Obs: Lettuce é compatível apenas com Python2.7-. Recomendamos substituir o Lettuce pelo Behave e utilizar Python3+ em um novo projeto (a diferença é sutil e pode ser encontrada .

# O que automatizar ?

Como esperado, começamos pelas atividades de maior relevância para nossa aplicação e que eram sempre reexecutadas para garantir que as principais funcionalidades continuavam funcionando, o chamado *Smoke Test*.

Após o *Smoke* começamos os teste de regressão, onde são adicionadas todas as funcionalidades já estáveis do sistema e que por questões de tempo, é praticamentes inviável de ser executado de forma manual. Com a regressão automatizada você consegue executá-la no fim do dia e ao retornar pela manhã obter os resultados ;)

Saiba que você não irá, e nem deve, automatizar todas as funcionalidades da sua aplicação, principalmente as que costumam sofrer mudanças recorrentes. Avalie os fluxos de seu sistema e saiba descartar o que não convém automatizar.

# É hora do código

Caso queira começar com um projeto exemplo antes de dar início ao seu, pode baixar meu [projeto no github](https://github.com/ladylovelace/little-monkey) que eu criei durante uma [palestra sobre automação de testes funcionais](https://speakerdeck.com/ladylovelace/building-your-own-monkey-the-era-of-automation) e que foi criado com base nos passos descrito abaixo.

## Passo 1: Organizando seu projeto

  
Para uma boa leitura e manutenção de código, o ideal é pensar em uma estrutura que te facilite no futuro. Utilizando BDD, e o framework lettuce, ele já nos força a ter pelo menos um diretório *features* . Dentro dele podem existir outros subdiretórios, como smoke e regression para guardar os arquivos *.feature* (Os cenários de teste em Gherkin) e src para o código python.

Em suma nossa estrutura ficou assim:

```
.
├── features
│   ├── regression
│   │   ├── cte-list.feature
│   │   ├── ...
│   │   └── nfe-list.feature
│   ├── smoke
│   │   └── smoke-test.feature
│   └── src
│       ├── config.py.dist (opcional/recomendado)
│       ├── steps.py
│       ├── routes.py (opcional)
│       └── terrain.py
├── README.md (opcional/recomendado)
└── requirements.txt (opcional/recomendado)
```

Para executar o projeto é preciso garantir que você possui todas as dependências instaladas (lettuce, selenium, chromedriver, ...). Para isto basta realizar a instalação destes pacotes através do [pip](https://pip.pypa.io/en/stable/).

```
pip install selenium
pip install lettuce
pip install chromedriver
```

Uma dica é utilizar o *virtualenv* para gerenciar e isolar estas dependências apenas ao teu projeto, evitando assim a necessidade de acesso de super usuário para instalação. Seu uso é simples e pode ser encontrador [aqui](https://virtualenv.pypa.io/en/stable/installation/).


### Arquivos de suporte
`features/src/config.py`
`features/src/config.py.dist`

No nosso caso, o arquivo *.config* serve para armazenar os dados de usuários da nossa aplicação, *paths* e o *browser* que vamos executar naquele momento. Isto nos permite maior reuso dos cenários de teste, privacidade de informações sensíveis (com o uso do arquivo *.dist*, senhas não são enviadas para o repositório) e o código torna-se mais limpo, facilitando a manutenção.

```python
# Hostname da aplicacao a ser testada
ARQUIVEI_HOST = 'https://qa.arquivei.com.br/'
# Catalogo de usuarios
# Exemplo de uso nos cenarios: Given I am logged in as "starter"
USERS = {
    'enterprise': {
        'username': 'enterprise@arquivei.com.br',
        'password': ''
    },
    'starter': {
    'username': 'starter@arquivei.com.br',
    'password': ''
    },
    ...
}
...
# Escolha o browser para realizar os testes
# Browsers disponiveis: chrome, firefox, headless
BROWSER = 'chrome'
```

`features/src/terrain.py`


Por convenção o *Lettuce* busca um arquivo chamado *terrain.py* no diretório. Pense nele como um arquivo de *setup* global. Ele contém funções úteis a toda execução do seu projeto como iniciar e fechar o browser e também encontrar um elemento na tela (função que vários steps utilizam).


```python
from lettuce import before, after, world

from selenium import webdriver
from selenium.common.exceptions import StaleElementReferenceException


@before.all
def setup_browser():
    if BROWSER == 'chrome':
        chrome_options = webdriver.ChromeOptions()
        chrome_options.add_argument('--no-sandbox')
        chromedriver = "/usr/bin/chromedriver"
        world.browser = webdriver.Chrome(executable_path=chromedriver,
                                         chrome_options=chrome_options)
        world.browser.set_window_size(1280,800)
    elif BROWSER == 'chrome-headless':
        chrome_options = webdriver.ChromeOptions()
        chrome_options.add_argument('--no-sandbox')
        chrome_options.add_argument('--disable-setuid-sandbox')
        chromedriver = "/node_modules/chromedriver/lib/chromedriver/chromedriver"
        world.browser = webdriver.Chrome(executable_path=chromedriver,
                                         chrome_options=chrome_options)
        world.browser.implicitly_wait(10)
        world.browser = webdriver.Chrome(executable_path=chromedriver,
                                         chrome_options=chrome_options)

@after.all
def tear_down_feature(self):
    world.browser.quit()


def contains_content(browser, content):
    for elem in browser.find_elements_by_xpath(unicode(
            u'//*[contains(normalize-space(.),"{content}") '
            u'and not(./*[contains(normalize-space(.),"{content}")])]'
            .format(content=content))):
        try:
            if elem.is_displayed():
                return True
        except StaleElementReferenceException:
            pass
    return False
```

## Passo 2: Escrever os cenários

Desde o princípio nós já escreviamos cenários de teste em [Gherkin](https://cucumber.io/docs/reference#gherkin), o que facilitou a portabilidade para a automação. É importante descrever os steps de forma clara, sucinta e reutilizável para evitar duplicidade de funções.

##### Obs: Os valores entre aspas servem como parâmetros das funções da automação. 

`features/smoke/smoke-test.feature`

```gherkin
Scenario: [23] Consult NFe - A1
  Given I am logged in as "enterprise"   
    And I go to "nfe-list"
  When I press button with "id" like "summary-consult-button"
  Then I should see "Consulta Resumo realizada"
```

## Passo 3: Escrever as funções para automação

As funções responsáveis por executar os casos de teste ficam organizadas dentro do arquivo *steps.py*.
Com conhecimento básico em [Python](https://python.org.br/introducao/) e com o auxílio da documentação do [Selenium Pyton](http://selenium-python.readthedocs.io/getting-started.html) você é capaz de criar seu código sem grandes problemas.
Outro grande auxiliar na construção das suas funções é o [Inspect do Chrome](https://developers.google.com/web/tools/chrome-devtools/inspect-styles/?hl=pt-br) que te mostra de maneira fácil os elemento do *DOM* de sua aplicação.

![automation-start-inspect]({{site.baseurl}}/assets/img/posts/automation-start-inspect.jpg){:class="img-fluid"}

Abaixo seguem as funções necessárias para a execução do cenário descrito no Passo 2:

`features/src/steps.py`

 ```python
from config import ARQUIVEI_HOST, USERS
from route import ROUTES
from lettuce import step, world
from selenium.webdriver.common.keys import Keys

 @step('I am logged in as "(.*?)"$')
def I_am_logged_in_as(step, user_id):
    with AssertContextManager(step):
        world.browser.get(urlparse.urljoin(ARQUIVEI_HOST, 'login'))
        user = USERS[user_id]
        world.browser.find_element_by_id('login-email').send_keys(user['username'])
        world.browser.find_element_by_id('login-password').send_keys(user['password'])
        world.browser.find_element_by_id('login-submit').send_keys(Keys.ENTER)

@step('I go to "(.*?)"$')
def go_to(step, url):
    with AssertContextManager(step):
        route = ROUTES[url]
        world.browser.get(urlparse.urljoin(ARQUIVEI_HOST, route['url']))

@step('I press button with "(.*?)" like "(.*?)"$')
def press_with(step, attr, value):
    selector = '[%s="%s"]' % (attr, value)
    button = world.browser.find_element_by_css_selector(selector)
    button.click()

@step('I should see "([^"]+)"$')
def should_see(step, text):
    assert_true(step, contains_content(world.browser, text.encode('utf-8')))
 ```

##### Note que **@step** possui a mesma descrição que os passos descritos em *smoke-test.feature* .

## Passo 4: Execução

Com o [lettuce](http://lettuce.it/tutorial/simple.html) a execução torna-se bem simples, basta seguir os comandos listados:

```sh
# Considerando que você está na raiz do projeto

# Para executar todos os testes:
$ lettuce

# Para executar os testes de uma funcionalidade:
$ lettuce features/smoke

# Para executar os testes de uma única feature
$ lettuce features/regression/cte-list.feature

# Para executar alguns cenários de uma feature
$ lettuce features/regression/cte-list.feature -s 1,3
```

## Dificuldades encontradas / Lessons Learned

Listarei abaixo os problemas mais comum que enfretamos e como resolve-los.

### A automação inicia porém não encontra o elemento na tela

Verifique se o elemento que procura não esta demorando para aparecer na tela. Para resolver este tipo de problema você deve trabalhar com as funções [wait](http://selenium-python.readthedocs.io/waits.html), ou seja, esperar X segundos ou até que o elemento Y fique visível na tela para então executar a sua ação. Tenha em mente que até a velocidade da internet pode impactar em seus testes, e um dia seu elemento pode demorar 1 segundo para aparecer e no outro mais de 5.

```python
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(driver, 10)
element = wait.until(EC.element_to_be_clickable((By.ID, 'someid')))
```

### Compatibilidade de versões
Seus testes funcionavam e não funcionam mais?

Já enfrentamos problemas com atualizaçôes do Chrome, do chromedriver e do selenium. É comum que versões atuais do Browser aceitem apenas versões mais recentes dos drivers (chromedriver).
Uma das vantagens de se usar virtualenv ou melhor ainda, criar um projeto em Docker te ajuda a manter a compatibilidade do seu projeto mais facilmente.

Quando seu projeto estiver maduro, o mais recomendável é que ele rode em diversos *browsers* , pois isto é uma das grandes vantagens da automação. Mas enquanto estiver iniciando recomendo que foque em um *browser* (o mais utilizado por seus clientes) e depois comece a garantir a compatibilidade para outros também.

### Headless vs Não Headless

Em teoria, a execução entre *headless* e não *headless* deveria gerar os mesmos resultados. Porém, na vida real, um cenário pode falhar apenas em modo *headless* .

Durante a execução da sua *Feature* o Lettuce vai te informando sobre falhas e sucesso o que facilita que você encontre o problema.

![automation-start-debug]({{site.baseurl}}/assets/img/posts/automation-start-debug.png){:class="img-fluid"}

Uma das vantagens de usar browser *headless* é que você pode iniciar os testes e realizar outras atividades em paralelo, sem que isto afete a automação. Mas o maior ganho do *headless* é permitir que a automação seja executada em qualquer servidor sem Display, permitindo o uso de ferramentas de integração contínua.

Existem também estratégias de *Debug* para browsers *headless* como o uso da biblioteca [pyvirtualdisplay](http://pyvirtualdisplay.readthedocs.io/en/latest/) que te permite tirar *screenshots* do momento antes/após a falha do teste.