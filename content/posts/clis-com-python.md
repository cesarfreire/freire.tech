---
title: "Construindo CLI's glamourosos no Python 💅✨"
date: 2023-12-08T13:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags: ["CLI", "Python", "Rich"]
author: "César"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Conheça ferramentas para deixar suas aplicações em linhas de comando extremamente estilosas."
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/cesarfreire/freire.tech/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---
## Introdução

Existem inúmeros pacotes e frameworks para construir aplicações em linha de comando de forma fácil e bonita.

Neste post iremos abordar algumas ferramentas que utilizo e construir uma pequena aplicação de linha de comando de exemplo.

## Let's code

Começando do início: vamos criar uma nova pasta em algum local de sua preferência.

```shell
mkdir my_cli
cd my_cli
```

Agora vamos iniciar um novo virtual environment, para isolar nossa aplicação e os pacotes instalados do restante da instalação padrão do Python. Você pode aprender mais sobre os venvs do Python na documentação oficial do VENV.

```shell
python3 -m venv .venv
```

A seguir, precisamos ativar o venv criado:

```shell
source .venv/bin/activate
```

Para ter certeza de que o seu venv está ativo, verifique se há uma observação entre parênteses no seu shell, como o exemplo abaixo:

![Venv ativo](/clis_python/venv.png "Venv ativo")

O próximo passo é instalar uma biblioteca própria para a criação da nossa aplicação CLI: Typer. Essa biblioteca é extremamente fácil e versátil.

Para instalar, digite o comando abaixo:

```shell
pip install "typer[all]"
```

Agora vamos iniciar nosso CLI. Crie um arquivo chamado main.py:

```shell
touch main.py
```

Abra-o com algum editor de código de sua preferência. No meu caso, utilizo o PyCharm.

Adicione o seguinte conteúdo no arquivo:

```python
import typer

app = typer.Typer()


@app.command()
def primeiro_comando():
    print('Olá! Esse é meu primeiro comando.')


@app.command()
def segundo_comando():
    print('Olá! Esse é meu segundo comando.')


if __name__ == '__main__':
    app()
```

Resumindo o que estamos fazendo:

- `import typer`: Importando a biblioteca que instalamos.
- `app = typer.Typer()`: Criando um objeto app do tipo Typer.
- `@app.command()`: Adicionando um comando ao meu objeto app. No nosso caso, adicionamos dois comandos. O comando executa a função onde o decorator está: `primeiro_comando` e `segundo_comando`

Executando nosso arquivo, iremos obter um erro:

![Erro ao executar o código](/clis_python/image-2.png "Erro ao executar o código")

Observe que, com poucas linhas, nosso código já entende que há dois comandos disponíveis e que nenhum deles foi informado na execução. Tudo isso graças ao Typer.

Agora execute novamente passando a opção --help como parâmetro:

![Comando com o help](/clis_python/image-3.png "Comando com o help")

O Typer nos gera essa orientação automaticamente: Qual a forma correta de utilizar o arquivo, as opções e também os comandos disponíveis.

Notou a interface colorida? O Typer utiliza a biblioteca Rich, que deixa o retorno para o usuário muito mais bonito.

Executando um de nossos comandos:

![Executando um comando](/clis_python/image-4.png "Executando um comando")

E temos o retorno que adicionamos na função.

---

## Definindo nossa aplicação

Para este exemplo, iremos construir um CLI que consulta uma API pública e retorna fatos aleatórios sobre gatos ou cachorros (sim, muito útil).

Para consultar API’s externas, utilizaremos o pacote chamado [HTTPx](https://www.python-httpx.org/).

Para termos uma opção diferente no uso das opções em nosso CLI, vamos também utilizar a biblioteca [Inquirer](https://pypi.org/project/inquirer/).

Instale os pacotes:

```bash
pip install httpx inquirer
```

Vamos agora criar um arquivo para fazermos as requisições para as API’s de onde virão os fatos sobre os gatos e cachorros. Vou chamar de api.py:

> Lembrando que as API's utilizadas aqui podem não estar funcionando mais...

```python
import httpx


def get_random_dog_fact():
    try:
        response = httpx.get('https://dog-api.kinduff.com/api/facts')
        response.raise_for_status()
        fact = response.json()
        return fact.get('facts', '')[0]
    except Exception as e:
        print(e)


def get_random_cat_fact():
    try:
        response = httpx.get('https://cat-fact.herokuapp.com/facts/random?amount=1')
        response.raise_for_status()
        fact = response.json()
        return fact.get('text', '')
    except Exception as e:
        print(e)
```

Lembre-se: aqui utilizamos duas API’s para exemplo, mas essas funções que são executadas podem ser qualquer outra rotina que você faz normalmente. Agora vamos alterar nosso comando para fazer algo útil: buscar os fatos.

Renomeie a de função primeiro_comando para buscar, e chame a função da API para ter certeza de que está funcionando:

```python
import typer
import api

app = typer.Typer()


@app.command()
def buscar():
    facts = api.get_random_dog_fact()
    print(facts)


@app.command()
def segundo_comando():
    print('Olá! Esse é meu segundo comando.')


if __name__ == '__main__':
    app()
```

Ao executar nosso CLI, iremos printar na tela o retorno da API:

![Execução com API](/clis_python/image-5.png "Execução com API")

Lembre-se: a API retorna fatos aleatórios de gatos e cachorros, portanto a saída deve ter sido com uma saída diferente da imagem acima.

Vamos melhorar nosso comando para permitir que o usuário escolha entre cachorro ou gato, utilizando o Inquirer.

```python
import typer
import inquirer
import api

app = typer.Typer()


@app.command()
def buscar():
    question = [
        inquirer.List('animal',
                      message="Sobre qual animal você quer saber um fato?",
                      choices=['Cachorros', 'Gatos'],
                      ),
    ]
    answers = inquirer.prompt(question)

    if answers.get('animal') == 'Cachorros':
        facts = api.get_random_dog_fact()
    elif answers.get('animal') == 'Gatos':
        facts = api.get_random_cat_fact()
    else:
        facts = "Escolha uma opção válida!"

    print(facts)


@app.command()
def segundo_comando():
    print('Olá! Esse é meu segundo comando.')


if __name__ == '__main__':
    app()
```

Execute novamente, e veja:

![Permitindo escolhas](/clis_python/image-6.png "Permitindo escolhas")


O Inquirer gerou uma lista com as opções que informamos no código, e permitiu que o usuário utilize as setas do teclado para selecionar qual opção é a desejada. Ao selecionar e teclar enter, um determinado trecho de código é executado pelo if/else.

Selecionando as opções, as chamadas da API são feitas:

![Retorno das escolhas](/clis_python/image-7.png "Retorno das escolhas")

Agora, vamos estilizar mais ainda com o [Rich Progress](https://rich.readthedocs.io/en/stable/progress.html):

```python
import typer
import inquirer
from rich.console import Console
from rich.progress import Progress, SpinnerColumn, TimeElapsedColumn

import api

app = typer.Typer()
console = Console()


@app.command()
def buscar():
    question = [
        inquirer.List('animal',
                      message="Sobre qual animal você quer saber um fato?",
                      choices=['Cachorros', 'Gatos'],
                      ),
    ]
    answers = inquirer.prompt(question)

    with Progress(
            SpinnerColumn(),
            *Progress.get_default_columns(),
            TimeElapsedColumn(),
            console=console
    ) as progress:
        if answers.get('animal') == 'Cachorros':
            task1 = progress.add_task(
                description="[cyan]Buscando fatos de cachorros...",
                total=None,
            )
            facts = api.get_random_dog_fact()

            progress.update(
                task1,
                completed=1,
                total=1,
                refresh=True,
                description="[green]Buscando fatos de cachorros... OK"
                if facts
                else "[red]Buscando fatos de cachorros... FALHA!",
            )
        elif answers.get('animal') == 'Gatos':

            task2 = progress.add_task(
                description="[cyan]Buscando fatos de gatos...",
                total=None,
            )

            facts = api.get_random_cat_fact()

            progress.update(
                task2,
                completed=1,
                total=1,
                refresh=True,
                description="[green]Buscando fatos de gatos... OK"
                if facts
                else "[red]Buscando fatos de gatos... FALHA!",
            )
        else:
            facts = "Escolha uma opção válida!"

        print(facts)


@app.command()
def segundo_comando():
    print('Olá! Esse é meu segundo comando.')


if __name__ == '__main__':
    app()
```

Ao chamar a API, como temos que aguardar o retorno, é exibido uma barra de progresso animada e bonita:

![Barra de progresso](/clis_python/image-8.png "Barra de progresso")

Assim que a chamada é concluída, o fato é impresso na tela e a barra é alterada para verde, para indicar o sucesso da execução:

![Barra de progresso verde](/clis_python/image-9.png "Barra de progresso verde")

Existem inúmeras customizações e configurações do Rich, você pode consultar a documentação oficial para mais detalhes.

Por hora temos um CLI com apenas um comando, que depende da escolha do usuário para executar determinadas ações.

O próximo passo é a compilação do pacote para ser instalado em outros locais, mas ficará para um próximo post.

tchau! =)