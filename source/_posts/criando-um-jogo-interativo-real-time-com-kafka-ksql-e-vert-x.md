---
title: Criando um jogo interativo "real-time" com Kafka, kSQL e Vert.x
date: 2022-12-16
tags: []
---

# Criando um jogo interativo "real-time" com Kafka, kSQL e Vert.x

Buscando um contato inicial com kSQL e Vert.x assim como entender otimizações e limitações dessas tecnologias, lembrei de uma demo por Aaron Lee em uma talk "Coding to Be Event-Driven” em que ele usou uma corrida de barcos em um game "click-fast”. Achei uma ótima ideia adaptar as tecnologias e mudar um pouco o contexto (e fazer com carros no lugar de barcos). Nesse sentido, este artigo pretende introduzir sobre kSQL e Vert.x e como foram aplicados no projeto.

Antes, vamos à ideia! O conceito é bem simples. Ao acessar a sessão do game, você é associado a um time (RED ou GREEN) e quando o host der início à partida seu dever — pelo time — é clicar no botão "Clique!” em 20 segundos. Ganha o time com mais cliques.

![](./images/cars-race-demo.png)

O objetivo prático do game é simples: contar cliques e associar estes cliques a times. Alguns desafios (ou requisitos) inclusos são:
- Pontuações em tempo real: todos participantes acompanham os carros se moverem e a contagem de cliques sendo atualizada durante a partida, instigando mais competição e mais interação.
- Times balanceados: conforme os participantes entram na sessão os times são designados de forma a balancear a quantidade entre os times
- Início coordenado: os participantes devem começar a clicar no mesmo momento, e durante 20 segundos. Para isso o sistema deve comunicar todos os participantes sobre o início da partida no mesmo momento.

Da forma mais simples, podemos ter um processo todo em memória em que duas variáveis `score_red` e `score_green` são incrementadas e consultadas pelos participantes. Esta foi a minha implementação inicial para ter o projeto já funcionando. No entanto, introduzindo algumas ferramentas de stream pode dar uma maior poder de escalabilidade e interatividade do game.

## Vert.x

Vert.x é um toolkit com um conjunto de APIs Java para construir aplicações reativas que são escaláveis e resilientes. Os principais fatores para optar neste projeto foram a forma de abstração que o Vert.x dá para otimização de uso dos recursos — dá para desenvolver sem ter tanta preocupação com threading e concorrência — e estrutura modular que permitiu montar os componentes necessários com pouca configuração e dependências. Além de funcionalidades já prontas para integrar WebSocket com Event Bus, clients de Kafka, etc. — que devo comentar melhor adiante.

Um conceito interessante do Vert.x é o de “Verticles” que se define como um deployment "actor-like” e modelo de concorrência. Dá para pensar como sendo uma thread (ou um objeto remoto) acessível por eventos.

Aqui utilizei verticles para representar as sessions, ou seja, cada partida do game. Basicamente quando o usuário cria uma partida, uma verticle é deployada com as informações e o seu ciclo de vida (pontuações, status, time vencedor, etc).

```java
public class CarRaceVerticle extends AbstractVerticle {
  private int greenScore = 0;
  private int redScore = 0;
  private CarRaceStatus status;

  public CarRaceVerticle(String sessionCode) {
    this.sessionCode = sessionCode;
    this.startKey = UUID.randomUUID().toString();
    this.status = CarRaceStatus.LOBBYING;
  }

  @Override
  public void start() throws Exception {
    // Faz subscribe nos eventos desta sessão com handlers assignGuest, bookClick, startSession
  }

  private void assignGuest(String guestId) { ... }
  private void bookClick(String guestId) { ... }
  private void startSession() throws InterruptedException { ... }
}
```

Como a comunicação é dirigida por eventos o Vert.x possui o Event Bus, uma API com semântica pub/sub em que o verticles podem se inscrever ou publicar eventos em tópicos. É possível também conectar o Event Bus com o front-end através de Websocket, o que foi feito nesse projeto usando o SockJS, assim o front-end também consegue se inscrever e publicar eventos.

A conexão com o Kafka também foi feita por meio de um verticle contendo dois clients Kafka: um producer e um consumer. O producer para produzir registros de clicks no tópico Kafka e o consumer para consumir atualizações de pontuações.

![](./images/cars-race-diagram.png)

Quando o usuário clica no botão no front-end:

1. via Websockt (SockJS) é enviada uma mensagem com o time e ID da sessão
2. o verticle da sessão, que está inscrito no ID, recebe a mensagem e valida o status atual (que deve ser RUNNING) e envia para o verticle do Kafka
3. o verticle do Kafka envia a mensagem para o tópico Kafka "car_race_clicks”

## Kafka e kSQL

A próxima etapa é determinar de fato as pontuações de uma partida e notificar todos os participantes sobre as pontuações, preferencialmente próximo a tempo real. Para isso optei por uma organização bem simples usando o kSQL -— que nada mais é do que um banco de dados voltado para streams de dados com uma semântica SQL.

É possível criar streams a partir de tópicos Kafka e criar views materializadas (ou mesmo tabelas) com a agregação de dados dos streams.

Antes de entrar nas queries que criam a view para este projeto, vamos ver quais escolhas de otimização de produção e consumo no tópico Kafka foram usadas. Após poucos testes aqui optei por priorizar a latência — principalmente por ser uma demo e não ter escalabilidade em número de participantes por sessão.

### linger.ms = 0

Definir o linger em 0 basicamente implica que o cliente não fará buffer das mensagens antes de enviar para o broker. Tem o benefício de passar a mensagem adiante mais rapidamente, mas a custo de mais uso de rede.

### acks = 0

Acks controla o número de réplicas que precisam confirmar a durabilidade da mensagem enviada. Definir o acks como 0 desabilita este controle e o cliente considera a escrita sempre como bem sucedida. O risco aqui é de perder mensagens, o que não é crítico para este projeto já que não tem tanto impacto se alguns cliques forem “perdidos”.

### enable.auto.commit

Esta configuração faz com que o client faça commit nos offsets de mensagens de forma automática — basicamente informar para o broker que a mensagem foi consumida. É mais performático e recomendado fazer isso manualmente de acordo com a necessidade da aplicação. Nesta demo faz sentido ter habilitado. 

### fetch.min.bytes = 1

Definir, no consumer, o `fetch.min.bytes` para 1 indica para o broker que ele deve enviar mensagens assim que elas estiverem disponíveis. Fazer isso tem um ganho de latência mas com o lado negativo de maior quantidade de interações com o broker para troca de mensagens e metadados.

…

Os registros publicados no tópico `car_race_clicks` tem a seguinte estrutura, em JSON:

```json
{
  "session": "KaYjz",
  "team": "RED"
}
```

Indicando qual a sessão (partida) e qual o time (RED ou GREEN). Aqui utilizei o ID da sessão como chave do particionamento. O comando para criar a view materializada no kSQL é bem simples:

```sql
CREATE TABLE car_race_scores WITH (format='json') AS
  SELECT session, team, COUNT(team) as score FROM car_race_updates
  GROUP BY session, team
  EMIT CHANGES;
```

Primeiro agrupamos a query por sessão e time com o `GROUP BY session, team` para que a pontuação que vamos calcular seja única por time e por partida. O `COUNT(team) as score` define que a pontuação nada mais é do que a contagem de quantas vezes aquele time aparece no stream, naquela sessão.

Por fim, com o `EMIT CHANGES` indicamos que quando for atualizada, esta tabela emitirá eventos. Isto é importante para capturar a mudança de pontuação e atualizar o painel no front-end.

O verticle do Kafka, que também é um consumidor fica "inscrito” no tópico em que essas atualizações são criadas e quando há uma mudança de pontuação ele informa ao verticle da sessão em questão que informa a todos os participantes da sessão (tudo isso através do Event Bus e via broadcast, no caso de notificar a todos).

Dessa forma, de ponta a ponta, temos um projeto básico de um game interativo, escalável e próximo ao tempo real. É importante dizer que estas tecnologias e a forma de utilizá-las foram o que direcionaram o projeto e não o contrário — certamente dá para atingir uma melhor performance e/ou escalabilidade com outro conjunto de tecnologias.

Entender como tunar o Kafka para entregar com maior rapidez é muito importante quando usamos essa tecnologia e sempre vai depender do tipo de dados, cadência e uma série de fatores — não há uma configuração que sirva para tudo.

O Vert.x neste contexto de aplicação dirigida por eventos teve um ótimo fit, possibilitando criar objetos "atores” com seus ciclos de vida independentes e com gestão abstraída de threads, temporizadores e afins, de forma que não há, por exemplo, uma necessidade de desenvolver um orquestrador/controlador, já que há uma semântica pub/sub, ou tratar os eventos de forma stateless, com o conceito de verticles.

Todo o código envolvido no projeto estão disponível [neste repositório do GitHub](https://github.com/mrmorais/racing-cars-demo).
