---
title: Aplicação real-time com socket.io (parte 1)
date: 2017-03-22 14:53:39
tags: [Pig Web Dice, Node.JS, Socket.io]
---

# Aplicação real-time com socket.io (parte 1)
22 de março de 2017

Neste artigo utilizaremos a biblioteca [socket.io](https://socket.io) para desenvolver uma aplicação multiplayer "real-time". A ideia surgiu quando me foi apresentado um jogo de dados chamado: ["Pig Dice"](https://goo.gl/C5iXdo) como atividade de "Linguagem de Programação I". Mas, enquanto para a matéria eu vou desenvolver a solução em C++, aqui vou criar uma versão web com Node.JS.

Basicamente vou utilizar Express (inicialmente), Socket.io (para comunicar o servidor com os jogadores), React JS (para fazer o front end). Perceba que o servidor não vai persistir informações dos jogadores em um banco de dados. A ideia é apenas armazenas os estados em um array de players. Isso significa que no momento em que o servidor descer, todas as informações de score, users e sockets serão perdidas.

Hoje irei apenas criar uma base simples de conexões, para que nos próximos artigos eu possa implementar realmente a lógica do jogo. Essa base contém somente o socket.io e o servidor web rodando. As features iniciais são, portanto:
- Ao receber nova conexão:
  - Gerar um nome para o usuário e adiciona-lo ao conjunto de usuários.
  - Notificar todos os usuários sobre as mudanças no conjunto.
  - Dizer para o usuário adicionado qual o nome gerado para ele.
- Ao desconectar:
  - Remover o usuário desconectado do conjunto.
  - Notificar todos os usuários sobre as mudanças no conjunto.

Antes de desenvolver estas features, vejamos como funciona o Pig Dice:

## Pig Dice
Um jogo de dados para **duas** pessoas. Cada jogador deve acumular a maior quantidade de pontos possíveis. Os pontos são determinados pelo resultado de cada rodada e cada participante joga uma rodada de cada vez. O primeiro inicia a primeira rodada jogando um dado. Se o número que saiu no dado for 1, o jogador não acumula nenhum ponto e passa a vez para o próximo; se for diferente de 1, ele pode jogar o dado novamente e ir acumulando os pontos na rodada atual. Desde que não caia 1, o jogador pode decidir se para ou continua jogando e acumulando mais pontos (com a possibilidade de tirar 1 e não ganhar nada na rodada). Se escolher parar, os pontos da rodada somam-se aos seus pontos totais. Ganha o jogo o primeiro jogador que fizer 100 ou mais pontos.

Se a explicação não ficou muito clara, recomendo ler sobre o jogo no Wikipédia: [Pig Dice](https://goo.gl/C5iXdo)

## Código no GitHub
O código da aplicação completa, ou seja, que será incrementada a cada artigo está disponível [neste link para o GitHub](https://github.com/mrmorais/pig_web_dice). No entanto, para haver um sentido atemporal na leitura dos artigos, cada novo código desta série estará no repositório [mrmorais/tutoriais](https://github.com/mrmorais/tutoriais)!

## Show me the code
[Neste artigo](http://mrmorais.com.br/2017/03/23/ecmascript-6-e-como-usar/) eu ensino como montar um setup ideal para Node JS com ES6, recomendo que antes de começar o desenvolvimento você o leia e monte o cenário necessário.

Crie um arquivo `app.js` na raiz do seu projeto, e vamos inicializa-lo para termos um servidor simples rodando.
> Note: se você não inicializou corretamente o setup necessário (NPM + Babel), nenhum código a seguir deverá funcionar. 

**{path}/app.js**
```javascript
import express from 'express';
import http from 'http';
import socketIo from 'socket.io';

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

app.listen(8080);
```

Importamos o `express`, framework web para Node que, neste momento, utilizaremos para entregar uma página HTML simples. Além disso também importamos o `http` que proverá o nosso servidor na porta 8080. E, obviamente, importamos o `socket.io` que será a base para a comunicação _event-based_.

Depois das importações, criei uma instância de express na `const app`. E passei esse app para ser servidor no `const server` que é criado na mesma linha de inicialização com o método `createServer` do `http`. Por fim, criamos uma instância do `socketIo` dizendo pra ele qual o servidor em que a comunicação com os usuários será estabelecida.

Não menos importante dizemos com o método `app.listen()` que o app deve escutar a porta 8080 e responder às requisições dela.

Com essas inicializações feitas, devemos pensar agora no gerenciamento dos usuários e fazer o controle das conexões. Como disse anteriormente, não vamos utilizar persistência em banco de dados, vamos armazenar as informações dos usuários em um array. Quanto ao controle, será feito pelo `io` que vai receber eventos e tratá-los. Para implementar isso, depois das inicializações (app, server, io) adicione o seguinte código:
```javascript
let guests = [];
let id_counter = 0;

io.on('connection', (socket) => {
  let _guest = { id: id_counter++, name: "mickey mouse", score: 0};
  guests.push(_guest);
  console.log(`>> New guest called ${_guest.name}`);
  console.log(`>> There are ${guests.length} guests online`);

  socket.on('disconnect', (data) => {
    let index = guests.indexOf(_guest);
    if (index > -1) {
      guests.splice(index, 1);
    }

    console.log(`>> ${_guest.name} disconnected`);
    console.log(`>> There are ${guests.length} guests online`);
  });
});
```

Vamos ver o que aconteceu aqui. Criamos um array chamado `guests` vazio, que receberá nossos jogadores. E também criamos um contador `id_counter` que irá ajudar na hora de identificar usuários.

A função `io.on` cria um listener para eventos lançados no socket. O primeiro argumento, neste caso `'connection'` indica o nome do evento que queremos tratar; o segundo argumento é uma função de callback. Aqui estamos utilizando a notação arrow function exclusiva do ES6. Um evento retorna para a gente um `socket` que é por onde nós iremos enviar e receber mensagens do usuário atrelado àquela conexão.

É simples entender: `socket` e `io` possuem o mesmo tipo de objeto, mas o `socket` lhe permite enviar e receber apenas de um usuário específico; o `io` permite fazer isso com todos os usuários conectados ao mesmo tempo. Veremos mais disso posteriormente.

Quando recebemos uma conexão, ou seja, um evento `'connection'` é lançado, ou seja, quando a função de callback é chamada, nós criamos um objeto para o usuário que se conectou.

```javascript
let _guest = { id: id_counter++, name: "mickey mouse", score: 0};
guests.push(_guest);
```

Perceba que `"mickey mouse"` será o nome genérico para todos os usuários e que quando setamos o id do usuário como `id_counter` também fazemos um incremento no `id_counter`, para que o próximo usuário adicionado tenha esse valor atualizado. Depois de criado, adicionamos o novo usuário à `guests` com o método `push()`. Depois de adicionado "logamos" a informação de que um novo usuário se conectou e quantos usuários estão online.

Adicionamos um novo listener, dessa vez é dentro do callback do `'connection'` e no lugar de usar o `io` estamos usando o `socket`. Lembre-se: `io` representa todos e o `socket` representa o atual.

O novo listener atende ao evento `'disconnect'` que, por acaso, é lançado quando o usuário conectado fecha a janela do browser, ou fica sem internet, ou infinitésimas coisas que podem ocasionar esse evento. Esse evento é importante para organizar-mos a lista de usuários onlines, se ele estariamos estocando apenas registro de usuários que passaram pelo site.

Dentro do callback para `'disconnect'` buscamos no array qual o índice do guest que desconectou, com a linha

```JavaScript
let index = guests.indexOf(_guest);
```

e depois retiramos o guest do `guests`, com o `splice()`, esse método faz um shift-left no array. Ou seja, pega todos os elementos a partir de certo `index+1` e sobre escreve o valor deles nas `n` casas à esquerda, aqui `n` é `1`. ([mais sobre splice](https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Reference/Global_Objects/Array/splice))

Esta foi uma aplicação simples, do lado do servidor, com um socket.io para receber, manejar conexões e "desconexões". O fluxo em sockets é essencialmente este: emitir (ainda não foi abordado) e receber dados e fluxos de dados.

Na próxima parte deste artigo, iremos desenvolver a aplicação do lado do cliente e veremos quais serão as implicações para o lado do servidor. Perceba que ainda não estamos enviando nada para o cliente, apenas recebendo.

Espero que este artigo tenha lhe sido útil. Vejo você no próximo. Lembre-se que os códigos deste artigo estão [neste repositório do GitHub](https://github.com/mrmorais/tutoriais)
