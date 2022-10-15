---
title: Retorno de processamento assíncrono com Redis e GraphQL Subscriptions
date: 2022-10-15
tags: []
---

# Retorno de processamento assíncrono com Redis e GraphQL Subscriptions

Não é difícil encontrarmos situações aplicáveis para processamentos assíncronos: processamento de pagamento, anti-fraude, validações com sistemas terceiros, etc. Em geral, são serviços que demandam integrações ou pode demorar tempo consideravelmente alto para uma requisição síncrona.

Neste artigo, usando o contexto de processamento de pagamentos, pretendo abordar uma forma de implementação de retorno em um processamento assíncrono. O foco, na verdade, é simular uma experiência síncrona, em que na mesma página o usuário esperaria o fim do processamento.

Certamente existem formas diferentes de obter um resultado parecido. Inicialmente podemos pensar em um *polling*, em que o cliente faria várias requisições para saber o status do processamento até estar finalizado. Apesar de ser de fácil implementação essa abordagem tem o ponto negativo pela quantidade de requisições necessárias, que pode ser grande se o processo for demorado.

## Implementando a demonstração

Criei um projeto bem simples que lista cada processo e seu status. O processo na verdade é um timer aleatório entre 0 e 10 segundos que finaliza com um status também aleatório “sucesso” ou “falha”.

![](/images/demo-async-proc.gif)

Para esta implementação vamos utilizar a feature de inscrição do GraphQL, o “subscription”, que nada mais é que uma abstração usando Web Socket. Uma sistema de pub/sub, como o Redis, nos permitirá notificar o retorno do processamento de forma desacoplada. Além deles, estou usando o ActiveMQ (que tem a mesma interface do SQS) como mecanismo de fila, que pode até mesmo ser substituído pelo Redis.

![](/images/async-process-comp.png)

É importante termos um identificador do processo (task id) já que o mesmo processo deve passar por diferentes sistemas e etapas. Aqui optei por **Client Generated ID**, um UUID gerado no front-end que passarei a chamar de “task id”.

A API GraphQL possui dois recursos: a mutation `initiatePayment` que inicializará o processamento e uma subscription `paymentProcessed` que será usada para escutar o retorno do processamento.

O front-end requisitará esses recursos quase ao mesmo tempo, dando início e já escutando o retorno da mensagem criada.

```graphql
type Mutation {
  initiatePayment(id: String!): PaymentProcess
}

type Subscription {
  paymentProcessed(id: String): PaymentProcess
}
```

### Iniciando o pagamento

A mutation é bem simples. Ela envia para uma fila SQS as informações do processo criado.

<script src="https://gist.github.com/mrmorais/d12385ce79510fa96f55ff8f4f40c6fe.js"></script>

Esta fila, por sua vez, é consumida pelo “Payment Processor” que gera os resultados aleatórios. O resultado do processamento é publicado no Redis Pub Sub em um tópico com nome “PAYMENT_PROCESSED.[task-id]”. Isso é o que tornará possível uma escuta reativa a este resultado, pois o cliente estará inscrito neste tópico.

<script src="https://gist.github.com/mrmorais/b112a204b652d5edaf9a8b7875307615.js"></script>

### Aguardando o retorno

Para que o front-end saiba que o pagamento foi processado e conheça o resultado desse processamento ele se inscreve na subscription “paymentProcessed” passando o task-id. A implementação desta subscription é tão simples que tem praticamente uma linha:

<script src="https://gist.github.com/mrmorais/e612a63948984e7441adcdb9859cf1d8.js"></script>

O método `asyncIterator` do [graphql-redis-subscriptions](https://github.com/davidyaha/graphql-redis-subscriptions) cria uma inscrição no Redis ao tópico informado e escuta os eventos publicados neste tópico, retornando um evento quando ele existir. O nome do tópico é o mesmo que o processador utilizará quando finalizar a execução.

Dessa forma o handler da subscription será chamado no front-end e ele reagirá de acordo mostrando o resultado do processamento.

Os códigos utilizados nesta demonstração são bem simples e fáceis de reproduzir. Eles estão todos disponíveis no meu GitHub [neste endereço](https://github.com/mrmorais/async-process).