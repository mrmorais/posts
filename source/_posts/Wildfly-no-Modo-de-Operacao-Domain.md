---
title: Wildfly no Modo de Operação Domain  
date: 2019-11-09 19:29:00
tags: 
---

# Wildfly no Modo de Operação Domain 
22 de outubro de 2019

## Conceitos do Modo Domain

O modo domain é um dos dois modos de operação do Wildfly. Previamente tratamos sobre o modo “standalone”, em que uma instância do serviço é executada em um host (físico ou virtual). O modo de operação domain gerencia um grupo de instâncias executando em múltiplos hosts, e permite o gerenciamento centralizado de recursos, módulos, extensões. O **domain controller** (ou master) define tais políticas. [^1]

Para compreender o modo domain devemos nos voltar à dois arquivos de configuração principais: domain.xml que define uma série de políticas, e server-groups e o arquivo host.xml, que define os servers que serão alinhados aos server-groups e à topologia em si.

Server-groups são definições compartilhadas de conjuntos de instâncias (processos Java) de profiles (full, ha, ha-full), parâmetros do JVM e grupos de socket bindings, que lida com as portas e interfaces associadas ao grupo. Enquanto o domain.xml define estas configurações, o host.xml define quais, e quantos, servers serão alocados para os grupos definidos.

No host.xml também é definido o tipo da instância no modo domain, isto é, se a instância é um master ou slave na topologia. Mais a seguir será demonstrada praticamente esta definição.

Resumindo, teremos três tipos de processos Java a considerar [^2]:

* Domain: processo Java, chamado domain controller (DC de agora em diante) que é o master, onde também fica disponibilizado o sistema de gerenciamento do domínio.
- Server group (SG): uma representação lógica das configurações do Wildfly. Composto por um ou mais instâncias de servidores.
- Host: processo Java, chamado host controller (HC), normalmente referido como processo slave. O DC se comunica com o HC publicando novas configurações.

A Figura 1 demonstra um panorama geral em que estes conceitos aparecem. Temos o DC e 3 HC, entre eles temos 3 SGs distribuindo 7 servers entre os hosts.

![Figura 1: Wildfly configurado em modo domain](/images/wildfly-server-group.png)

## Instalando e configurando o Wildfly

Para este tutorial será considerada a utilização de um software de virtualização, pois estaremos virtualizando 3 máquinas virtuais (um DC e dois HC). Para isto estou utilizando o Oracle Virtual Box com a imagem do sistema operacional CentOS. Também é importante uma pré-configuração da máquina com desabilitação do serviço firewalld e do SELinux. Também verifique ter o Java instalado. Com estes requisitos prontos, podemos dar início.

Vamos dar início com a instalação do Wildfly. Acessando o [https://wildfly.org](https://wildfly.org), na sessão Downloads, você encontrará facilmente o endereço para o download da versão Full mais recente. Aqui está sendo utilizada a versão 18:

```shell
# wget https://download.jboss.org/wildfly/18.0.0.Final/wildfly-18.0.0.Final.zip
# unzip wildfly-18.0.0.Final.zip
# mv wildfly-18.0.0.Final /opt/wildfly
```

Com os comandos acima baixamos, descompactamos e movemos o diretório do wildfly para o /opt/wildfly.

## Criando usuário e dando permissões

Vamos criar o usuário wildfly que terá a propriedade do diretório recém criado /opt/wildfly:

```shell
# useradd wildfly
# chown -R wildfly. /opt/wildfly/
```

## Criando o serviço de execução

Para ter uma melhor estrutura de execução do Wildfly é interessante configurar um Unit File no nosso systemd, assim podemos utilizar comandos mais elegantes para inicializar o serviço e habilitar para que seja iniciado com o host. Para nossa sorte o wildfly acompanha um template de arquivos que podem ser simplesmente copiados para os locais indicados que o serviço estará devidamente configurado, sem a necessidade de criar o Unit File manualmente.

Estes templates estão localizados em $WILDFLY_HOME/docs/contrib/scripts/systemd. Recomendo a leitura do README que se encontra também neste endereço. Vamos então copiar os arquivos:

```shell
# mkdir /etc/wildfly
# cp wildfly.conf /etc/wildfly/
```

Este **wildfly.conf** contém variáveis de ambiente que definem o modo de operação é a interface de binding do serviço, ele vem definido para utilizar o modo standalone. Vamos modificá-lo para modo domain fazendo as seguintes mudanças:

```
WILDFLY_CONFIG=domain.xml
WILDFLY_MODE=domain
```

Continuando, copie o restante dos arquivos:

```shell
# cp wildfly.service /etc/systemd/system
# cp launch.sh /opt/wildfly/bin
```

Agora temos o setup para inicialização do modo domain, vamos habilitá-lo para inicialização junto ao host e iniciar o serviço:

```shell
# systemctl enable wildfly
# systemctl start wildfly
```

Pode ser necessário reiniciar o workload do systemd, com o comando:

```shell
# systemctl daemon-reload
```

O endereço do host na porta 8080 deve retornar a tradicional página inicial do Wildfly.

A partir deste ponto temos a bifurcação na configuração do master e os slaves, ambos os tipos devem possuir as mesmas configurações que fizemos até o momento. Portanto, recomendo que, agora, desligue o host que está configurando e faça um clone do mesmo renomeando-o para “node1”, por exemplo, de forma que o atual será o nosso master.

## Configurando o host master

Como descrito na conceituação, os arquivos de configuração domain.xml e host.xml são os que devemos modificar para personalizar nosso modo domain. Eles se encontram em $WILDFLY_HOME/domain/configuration/.

Iniciando com o domain.xml, é onde se definem os server-groups (SGs), esta definição é feita na tag <server-groups>, vamos alterá-los da seguinte forma:

```
<server-groups>
    <server-group name="server-group-a" profile="full">
        <jvm name="default">
            <heap size="64m" max-size="512m"/>
        </jvm>
        <socket-binding-group ref="full-sockets"/>
    </server-group>
    <server-group name="server-group-b" profile="full-ha">
    	<jvm name="default">
	    <heap size="64m" max-size="512m"/>
	</jvm>
	<socket-binding-group ref="full-ha-sockets"/>
    </server-group>
</server-groups>
```

No arquivo host.xml temos algumas tags interessantes, a primeira delas é <domain-controler>:

```
<domain-controller>
    <local/>
</domain-controller>
```

Esta configuração, com a tag <local/>, indica que o host atual é o próprio DC (domain controller). Você verá que nos hosts slaves esta tag conterá uma referência ao DC.

No meu ambiente foi necessário também modificar as interfaces mudando de 127.0.0.1 para 0.0.0.0 nos endereços:

```
<interfaces>
    <interface name="management">
        <inet-address value="${jboss.bind.address.management:0.0.0.0}"/>
    </interface>
    <interface name="public">
        <inet-address value="${jboss.bind.address:0.0.0.0}"/>
    </interface>
</interfaces>
```

Note que para a interface “public” esta modificação não seria necessária pois já é alterada automaticamente pelo script launch.sh, mas a interface de management não é alterada pelo script e por isso tornou-se necessário definir o endereço manualmente no host.xml.

Também são definidos no host.xml os servers, na tag <servers>:

```
<servers>
    <server name="server-one" group="server-group-a"/>
    <server name="server-two" group="server-group-a" auto-start="true">
        <jvm name="default"/>
	<socket-bindings port-offset="150"/>
    </server>
    <server name="server-three" group="server-group-b" auto-start="false">
        <jvm name="default"/>
	<socket-bindings port-offset="250"/>
    </server>
</servers>
```

Com estas configurações, o nosso master está pronto para entrar em modo domain. Lembre-se de reiniciar o serviço com o systemctl. Vamos agora aos slaves:

## Configurando o Node 1

Lembre-se que anteriormente recomendei fazer o clone da configuração do master até certo momento. Nesta máquina virtual clonada daremos início à configuração do primeiro nó: Node 1.

No host do node1 vamos também modificar o arquivo host.xml. O host.xml vem definido como padrão para ser um master, mas já temos um master em execução. Felizmente existe também um arquivo host-slave.xml que é um template para configuração do slave. Vamos salvar o host.xml atual e utilizar o host-slave.xml no lugar:

```shell
# mv host.xml host.bkp
# cp host-slave.xml host.xml
```

Vamos primeiramente dar nome ao host modificando a tag host:

```
<host xmlns="urn:jboss:domain:10.0" name="node1">
```

Agora definiremos a referência para o master, para isso você deve modificar a domain-controller com o ip do master, no meu caso é 0 198.168.0.18:

```
<domain-controller>
    <remote security-realm="ManagementRealm" username="node1user">
    	<discovery-options>
	    <static-discovery
	        name="primary"
		protocol="${jboss.domain.master.protocol:remote+http}"
		host="${jboss.domain.master.address:192.168.0.18}"
		port="${jboss.domain.master.port:9990}"/>
	</discovery-options>
    </remote>
</domain-controller>
```

Note que na tag <remote> eu adicionei o atributo username. O que acontece aqui é que o node1 passará por um controle de acesso feito pelo master.

Para isso você deverá, no master, criar um usuário para acesso do node. Isso é feito com o script $WILDFLY_HOME/bin/add-user.sh:

```
What type of user do you wish to add?
a) Management User (mgmt-users.properties)
Username : node1user
Password :
Is this correct yes/no? yes
Is this new user going to be used for one AS process to connect to another AS process? ...
yes/no? yes
```

Note que ao final da execução ele retornará uma tag parecida com esta: <secret value="bm9kZTJ1c2Vy" />

Esta tag deve ser inserida numa tag server-identities no host.xml:

```
<security-realm name="ManagementRealm">
    <server-identities>
        <secret value="bm9kZTF1c2Vy"/>
    </server-identities>
    ...
```

Com estas configurações, o host slave estará pronto para ser acoplado ao domínio.

![Figura 2: Hosts, Server-groups e Servers na topologia criada](/images/wildfly-topologia.png)

A Figura 2 apresenta um panorama geral dos hosts (um master e dois slaves) seus servers e os server-groups criados ao final do tutorial.

[^1]: https://kb.novaordis.com/index.php/WildFly_Domain_Mode_Concepts
[^2]: https://static.packt-cdn.com/downloads/2413OS_Appendix.pdf
