---
title: "Executando scripts personalizados nas triggers - Zabbix"
date: 2024-02-09T11:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags: ["Zabbix", "Monitoring", "Scripts", "Bash"]
author: "César"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Como executar scripts externos ao disparar uma trigger no Zabbix."
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
    image: "/scripts_triggers_zabbix/zabbix.svg" # image path/url
    alt: "Zabbix Image" # alt text
    caption: "Zabbix" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/cesarfreire/freire.tech/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---
## Introdução

Algumas operações realizadas pela equipe de infraestrutura envolve não apenas ação humana, mas também o acompanhamento de algo para que determinada ação deva ocorrer. Este processo vem sendo facilitado por inúmeras ferramentas, que apresentam soluções praticamente autônomas.

Em projetos novos, é comum nos depararmos com situações específicas em que a automatização de tarefas resolve um grande "buraco". Neste post, venho demonstrar como executar scripts a partir do disparo de triggers do Zabbix, que pode ser absolutamente qualquer coisa. Este processo envolve apenas as opções nativas do Zabbix, que é "for free" 💸💸💸.


## Let's do it

A primeira etapa é ter algo para observarmos. Vamos utilizar um Item dentro do Zabbix. Esse item pode ser qualquer coisa: alguma métrica de algum host, algum atributo e até alguma informação externa.

No exemplo, irei utilizar o seguinte exemplo:

- Monitorar uma URL (CNAME do host) e acompanhar o endereço de IP. Caso o IP altere, uma trigger é disparada e um script (comando externo) é executado.

É algo simples, mas pode ser bem útil.

Portanto, o item que irei utilizar é um script externo que faz a resolução _CNAME_ > _A_. Você pode aprender mais sobre os tipos de registros de DNS [nesse link](https://www.cloudflare.com/pt-br/learning/dns/dns-records/).

### Item

No servidor onde o Zabbix está em execução, normalmente os scripts externos ficam localizados no diretório `/usr/lib/zabbix/`.

Irei criar um script bash chamado `get_ip.sh`:

```shell
touch /usr/lib/zabbix/externalscripts/get_ip.sh
```

O conteúdo é um código bash simples, que busca o IP (registro A) de uma URL:

```bash
#!/usr/bin/bash

# Verifica se foi fornecido um argumento
if [ $# -ne 1 ]; then
    echo "Usage: $0 <hostname>"
    exit 1
fi

# Armazena o hostname fornecido como argumento
host=$1

# Resolve o hostname para endereços IP e faz o sort
ips=$(dig +short $host | sort)

# Exibe os endereços IP encontrados
if [ -z "$ips" ]; then
    echo "No IP addresses found for $host"
else
    # Substitui a quebra de linha (caso tenha mais de um IP) para vírgula
    echo "$ips" | tr '\n' ',' | sed 's/,$//'
fi
```

Não esqueça de aplicar as permissões necessárias para o usuário `zabbix` conseguir executar:

```bash
# torna o arquivo executável
chmod +x /usr/lib/zabbix/externalscripts/get_ip.sh

# altera o owner para o zabbix
chown zabbix:zabbix /usr/lib/zabbix/externalscripts/get_ip.sh
```

Agora, vamos configurar o item na interface do Zabbix. Você pode fazer a criação diretamente no host, porém, é interessante utilizar os templates do Zabbix, que pode ser adicionado em N hosts. Não demonstrarei nesse tutorial, mas verifique a [documentação sobre os templates](https://www.zabbix.com/documentation/current/pt/manual/config/templates/template).

Neste caso, nosso item irá utilizar o script `get_ip.sh`.

![Configuração do item](/scripts_triggers_zabbix/item1.png "Configuração do item")

> Repare que utilizei o tipo `External Check`, e a chave `get_ip.sh[{$HOST.CNAME}]`. O tipo retornado é `Text`, e o intervalo de coleta é a cada 2 minutos.

Basicamente, estamos dizendo que o nosso item vai ser o resultado da execução do script `get_ip.sh`, passando como parâmetro a propriedade `CNAME` do nosso `HOST`, que será o host associado com o template que criamos.

### Host

Agora, vamos criar o host e associar o template criado. Como adicionamos o script no próprio servidor Zabbix, utilize um host com o apontamento (seja o IP ou o CNAME) para o próprio server.

> Lembre de ter o zabbix-agent2 instalado, configurado e em execução na máquina do Zabbix server, assim você pode usar o localhost (127.0.0.1) e a porta 10050.

![Configuração do host](/scripts_triggers_zabbix/host1.png "Configuração do host")

Agora, precisamos também configurar a propriedade `HOST.CNAME`, que será utilizada no item. Faça a criação na aba `Macros`:

![Configuração do CNAME](/scripts_triggers_zabbix/host2.png "Configuração do CNAME")

Ao salvar, se tudo ocorrer bem, a coleta do IP será efetuada a cada 2 minutos. Você pode observar os dados na seção `Latest Data`.


### Trigger

Com o item sendo coletado corretamente, vamos criar a trigger, que é o gatilho para a execução do comando externo:

![Configuração do CNAME](/scripts_triggers_zabbix/trigger1.png "Configuração do CNAME")

> Estou utilizando uma expressão bem básica, que apenas detecta a mudança. Você pode pesquisar e criar expressões mais avançadas, se desejar.

Nesse momento, já estamos com o alerta funcional da mudança do IP. Vamos partir para a execução do script a partir desse alerta.

### Scripts

Nosso objetivo é executar um script (comando externo) assim que a trigger for disparada. Precisamos primeiro cadastrar esse comando na aba "Scripts" do nosso servidor Zabbix (Administration > Scripts). No meu caso, executarei um script em Python que atualiza as rotas de alguns servidores de VPN, utilizando o IP do host.

> Lembre-se: Este comando pode ser absolutamente qualquer coisa, dependendo da sua necessidade. Não irei disponibilizar o script que utilizo pois não é o objetivo deste post.

![Configuração do comando](/scripts_triggers_zabbix/script1.png "Configuração do comando")

> Importante: Para a execução automática, selecione o escopo `Action Operation`.

### Trigger Action

Para finalizamos, vamos criar a ação da trigger, que vai ser executada no disparo. Acesse pelo caminho Configuration > Actions > Trigger actions. Crie uma nova Action e adiciona a trigger criada como condição:

![Configuração da trigger action](/scripts_triggers_zabbix/actiontrigger1.png "Configuração da trigger action")

Na aba Operations, adicione uma operação selecionando o script que criamos anteriormente:

![Configuração da trigger action](/scripts_triggers_zabbix/actiontrigger2.png "Configuração da trigger action")

No Target list, marque a opção `Current host`.

E pronto! Assim que o item (CNAME da URL) monitorado mudar de IP, a trigger irá disparar o script com o comando.

tchau! =)