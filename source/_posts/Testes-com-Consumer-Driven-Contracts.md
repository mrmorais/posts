---
title: Testes com Consumer-Driven Contracts
date: 2019-10-22 11:12:12
tags:
---

# Testes com Consumer-Driven Contracts
22 de outubro de 2019


No desenvolvimento de micro-serviços, por exemplo, é importante a existência de uma cultura de princípios guiando o processo de desenvolvimento. Um destes princípios é a **descentralização**: todos os times deve possuir controle e liberdade sobre seus serviços (desde o provisionamento de recursos ao deploy independente). Um ponto importante para garantir descentralização é a coordenação entre serviços, que devem ser o mais desacoplados possível. Um serviço deve responder de forma consistente para que outros serviços possam utilizá-lo apropriadamente, mas em ambientes de constantes modificações esta coordenação entre equipes pode ser falível e demandar testes de integração complexos de e2e (*end-to-end*).

<details>
<summary>Definição de testes e2e</summary>
Testes e2e (de ponta a ponta) são testes de mais alto nível e que exigem ambiente com todo o sistema (ou uma parte) em execução, eficaz em testar lógicas de negócios pensando no usuário.<br/>
</details><br/>

Como é possível ter a segurança de que serviços inter-dependentes não sejam negativamente impactados com mudanças? Esta é a proposta dos testes baseados em contratos com consumidores (ou **Consumer-Driven Contracts**). Este padrão garante a consistência nos relacionamentos entre aplicações ao efetuar testes nas APIs impondo a conformidade com um contrato que é estabelecido pelas expectativas das aplicações que consomem as APIs.

Logo, temos dois tipos de aplicações envolvidas:

- **Consumidor** (*consumer*): aplicação que depende e faz utilização de recurso de outro serviço, seja através de HTTP ou mensageria.
- **Provedor** (*provider*): serviço fornecedor de recursos que tem o consumidor como dependente e que atende via API às requisições.

A relação entre estas aplicações é definida por uma forma de contrato, um documento que especifíca as interações existentes: quais requisições o consumidor faz e quais respostas são esperadas do provedor para essas requisições. Ou seja, o esquema do contrato é definido com base em exemplos positivos.

O consumidor define como espera ser respondido (momento de criação do contrato.) Quando são feitas alterações no código do serviço provedor, o teste de validação informa se o contrato foi quebrado em decorrência da alteração, interrompendo o progresso da build na pipeline de deploy, por exemplo.

Esta prática possui benefícios práticos:

* Esconde a implementação: os times não precisam conhecer nenhum trecho de código de um serviço de outro time para entender como a resposta é gerada ou como é esperada, tudo fica a cargo de satisfazer o contrato.

- Cria um elo de confiança: podem ser feitas alterações no provedor sem medo de que outras aplicações sejam prejudicadas (a seguir veremos que, com uma ferramenta de broker de contratos, o provedor nem mesmo precisa conhecer quais aplicações dependem dele).

Certamente um dos maiores benefícios é o desacoplamento entre as aplicações uma vez que as interfaces são mais bem definidas.

Neste artigo irei definir esta prática nos termos do framework [Pact](pact.io), uma ferramenta de *contract tests*; provavelmente a mais utilizada para este fim. O ecossistema de Pact inclui também o [Pact Broker](https://docs.pact.io/pact_broker), aplicação para compartilhamento de contratos e resultados de verificações.

Estarei utilizando uma aplicação desenvolvida por mim com base no exemplo do pact-workshop[^repo-pact-workshop] em Javascript, mas Pact possui implementações e guias para diferentes linguagens.

## Funcionamento do Pact

Pact atua como *mock* nos dois lados da interação: para o *consumer* ele faz o papel do *provider* e o contrário para o *provider*. O ponto de partida é a especificação no lado *consumer*. O Pact intercepta as requisições que iriam para o *provider* respondendo com o que o *consumer* define no Pact test como esperado, neste momento os testes unitários do *consumer* também são realizados para garantir que com a resposta esperada do *provider* ele saberá operar como esperado também.

Ao final dos testes no lado *consumer*, se todos os teste passam, é gerado um arquivo `.json` com um conjunto de interações (especificações de requisição e resposta esperada). Este arquivo é, por assim dizer, o pacto (contrato) do relacionamento, o *provider* precisará somente dele para validar se está de acordo com as especificações do *consumer*. O pacto deve ser distribuído para que o *provider* tenha acesso. Existem várias formas de fazer esta distribuição: transferir via diretório compartilhado, repositório git ou, o mais recomendado, um Pact Broker (que é abordado na sessão [Pact Broker](#pact-broker)).

A figura abaixo dá uma visão geral do funcionamento do Pact.

<br/>

![](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LC2AYrI9MJa-_aAjE1u%2F-LN88wE6mKsXwlpUIcxu%2F-LN88wz8GOdj0witaWgd%2Fpact-test-and-verify.png?generation=1537751898934036&alt=media)
<center><small>Panorama geral dos componentes sob teste em Pact. Fonte: https://docs.pact.io/getting_started/how_pact_works</small></center><br/>

## Aplicação demonstrativa

A aplicação exemplo no lado *consumer* passa no corpo da requisição uma lista de itens (descrições, preços e quantidades) para o *provider* que, então, calcula o valor total de uma compra com tais itens e define uma porcentagem máxima de desconto aplicável. O *consumer* adiciona uma restrição extra na porcentagem de desconto limitando-o a 15%.

<center>consumer.js</center>

```js
const checkoutOrder = items => {
    return request
        .post(`${apiUri}/checkout`)
        .send({
            items
        })
        .then( res => {
            const {totalAmount, maxDescount} = res.body;
            const descount = maxDescount <= 0.15 ? maxDescount : 0.15;
            return {
                totalAmount,
                descount: 0.15,
                withDescountAmount: (1 - descount) * totalAmount
            };
        })
};
```

No lado do provedor o tratamento para a rota de checkout é calcular o valor total da ordem fazendo uma redução no array de itens recebido.

<center>provider.js</center>

```js
server.post('/checkout', (req, res) => {
    const items = req.body.items;

    const total = items.reduce((sum, currentItem) => 
        sum + (currentItem.price * currentItem.quantity)
    , 0);
    
    res.json({
        totalAmount: total,
        maxDescount: 0.2
    });
})
```

### Testes no lado *Consumer*

O teste está definido utilizando o framework Mocha com Chai. O Pact entra inicialmente como mock do serviço *provider*. É feita a importação e a definição do Pact em que são especificados nomes, portas e arquivos em que serão realizados outputs de logs e do pacto.

<center>consumerPact.spec.js</center>

```js
const Pact = require('@pact-foundation/pact').Pact;

const provider = new Pact({
    consumer: 'My consumer',
    provider: 'My provider',
    port: 3000,
    log: path.resolve(process.cwd(), 'logs', 'pact.log'),
    dir: path.resolve(process.cwd(), 'pacts'),
    logLevel: 'warn',
    spec: 2
});
```

No corpo do teste esta definida a interação "a request for total amount" que define o formato da requisição (`withRequest`) e a resposta que o mock criado pelo Pact irá dar ao *consumer* (`willRespondWith`). Em seguida é feita a validação da resposta, verificando se o `totalAmount` é igual ao valor esperado (156.5) e se o valor final com desconto calculado pelo *consumer* é também o esperado (133.025).

As validações são feitas sobre os resultados provenientes do *consumer* pois é justamente o que se deseja testar, se o *consumer* consegue prover suas funcionalidades adequadamente com os retornos consistentes vindos do *provider*.

<center>consumerPact.spec.js</center>

```js
describe('and passing a valid items set', ()=> {
    before(() => {
        return provider.addInteraction({
            uponReceiving: 'a request for total amount',
            withRequest: {
                method: 'POST',
                path: '/checkout',
                body: { items },
                headers: {
                    'Content-Type': 'application/json'
                }
            },
            willRespondWith: {
                status: 200,
                headers: {
                    'Content-type': 'application/json; charset=utf-8',
                },
                body: {
                    totalAmount: 156.5,
                    maxDescount: 0.2
                }
            }
        });
    });

    it('can process the JSON payload from provider', (done) => {
        const processedOrder = checkoutOrder(items);

        expect(processedOrder).to.eventually.have.property('totalAmount', 156.5);
        expect(processedOrder).to.eventually.have.property('withDescountAmount', 133.025).notify(done);
    });

    it('should validate and create a contract', () => {
        return provider.verify();
    })
});
```

Quando finalizada a execução dos testes, o Pact cria, no diretório definido anteriormente, o arquivo com as especificações. Este arquivo de pacto deve ser distribuído para o *provider*. Este projeto utiliza [Pactflow](https://pactflow.io/), ferramenta online de Pact Broker para distribuição de pactos. É possível, também, executar Pact Broker em um servidor próprio[^pact-broker].

<center>my_consumer-my_provider.json</center>

```
{
  "consumer": {
    "name": "My consumer"
  },
  "provider": {
    "name": "My provider"
  },
  "interactions": [
    {
      "description": "a request for total amount",
      "request": {
        "method": "POST",
        "path": "/checkout",
        "headers": {
          "Content-Type": "application/json"
        },
        "body": {
          "items": [
            {
              "description": "Antique typewriter",
              "price": 49,
              "quantity": 1
            },
            {
              "description": "Christmas decorations",
              "price": 15,
              "quantity": 5
            },
            {
              "description": "Notepad",
              "price": 16.25,
              "quantity": 2
            }
          ]
        }
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-type": "application/json; charset=utf-8"
        },
        "body": {
          "totalAmount": 156.5,
          "maxDescount": 0.2
        }
      }
    }
  ],
  "metadata": {
    "pactSpecification": {
      "version": "2.0.0"
    }
  }
}
```

### Validação no lado *Provider*

O lado *provider* tem a responsabilidade de estar em conformidade com o contrato. O teste consiste em buscar a distribuição do arquivo de pacto `.json` e validar as interações no ponto de vista do *provider*. A especificação, portanto, deve inicializar uma instância do servidor *provider* e utilizar o `Verifier` do Pact em seguida.

Neste projeto, como está sendo utilizado Pactflow, as configurações incluem `pactBrokerUrl` e o `pactBrokerToken` que são as informações de conexão ao broker. Mas é possível configurar para utilizar o arquivo `.json` informando o caminho para o arquivo localmente. 

<center>providerPact.spec.js</center>

```js
describe('Provider test', () => {
    let serverConn;
    before(() => {
        serverConn = server.listen(3000, () => {
            console.log('Provider is running ...');
        });
    })
    describe('Matches \"My consumer\" requirements', () => {
        it('should validate the contract', () => {
            return new Verifier().verifyProvider({
                provider: 'My provider',
                providerBaseUrl: 'http://localhost:3000',
                pactBrokerUrl,
                pactBrokerToken
            }).then();
        })
    });

    after(() => {
        if (serverConn) serverConn.close();
    })
});
```

Executando os testes é possível saber se, em produção, a aplicação entrará ou não em conflito com o *consumer*. Isto possibilita uma grande liberdade na alteração de como as coisas são feitas internamente e mesmo como os retornos são realizados. Imagine que a função de checkout foi alterada e, por algum engano, o retorno acrescente 5 reais ao montante final. A execução do teste reclamaria que esperava um valor diferente.

```
 Diff                                           
 --------------------------------------           
  {                                             
 -  "totalAmount": 156.5                        
 +  "totalAmount": 161.5                        
  }                                             
                                                
 Description of differences                     
 --------------------------------------         
 * Expected 156.5 but got 161.5 at $.totalAmount
```

## Conteúdos relacionados

Grande parte do conteúdo deste artigo se baseia conceitualmente na publicação em vídeo "The Principles of Microservices" de O'Reilly Media apresentado por Sam Newman e no livro "Building Microservices" também do Newman (uma leitura recomendada).

Imagens e conceitos do framework Pact são do [Pact](pact.io).

[^repo-pact-workshop]: Repositório no Github: [pact-workshop-js](https://github.com/DiUS/pact-workshop-js)
[^pact-broker]: [Página do Pacto Broker](https://docs.pact.io/pact_broker)