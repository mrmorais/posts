---
title: C++eleste - Jogo da cobrinha autônomo
date: 2017-06-17
tags: []
---

# C++eleste - Jogo da cobrinha autônomo
17 de junho de 2017

Como no artigo [“Aplicação real-time com socket.io“](/Aplicacao-real-time-com-socket-io-pt-1/) (que por sinal é um projeto de estou devendo dar continuidade) foi proposto como atividade de avaliação de habilidades de “Linguagem de Programação I” o projeto “Snake”, que consiste em utilizar uma técnica conhecida como “backtracking” ([leia mais](https://pt.wikipedia.org/wiki/Backtracking)), para resolver um labirinto; mais que isso, adaptar este algoritmo para controlar um jogo da cobrinha.

O vídeo abaixo mostra o jogo em funcionamento. São carregadas várias fases do jogo de um arquivo com um formato padronizado, e a cobrinha tem a missão de, sozinha, encontrar 3 maças em cada fase.

<iframe width="560" height="315" src="https://www.youtube.com/embed/4H9tBkkyHjg" frameborder="0" allowfullscreen></iframe>

A inteligência que resolve o labirinto foi desenvolvida pelo meu colega Franklin Matheus, foi feita uma abstração entre o conceito "jogo da cobrinha" -> "solução de labirintos", uma vez que o “Snake” é um labirinto com algumas especialidades a mais, como: a cobrinha só pode seguir uma direção, a “saída” do labirinto é a maçã que a cobrinha busca. Com essas abstrações foi fácil utilizar um “maze solver” para resolver o problema do jogo da cobrinha. E isso ficou por minha responsabilidade, adaptar um solução de labirintos para resolver um jogo da cobrinha. Se aprofundando no código dessa belezura fica mais fácil entender essa adaptação, mas o meu ponto aqui não é mostrar o funcionamento interno, mas sim falar sobre o desenvolvimento da interface gráfica que foi minha maior responsabilidade no projeto.

A Celeste (esse nome é referência à cobra do castelo Rá-Tim-Bum) utiliza a biblioteca SFML ([leia mais](https://www.sfml-dev.org/)) Simple and Fast Multimedia Library. A intenção foi fazer com o jogo desprendido da interface gráfica em si, em que facilmente possamos criar outra interface (via terminal, por exemplo) por isso SFML e Celeste são coisas bem definidas e diferentes. Eles se comunicam através de containers de status. A interface chama o método step() da Celeste para que ela tome a próxima decisão (seja ela inicializar uma partida, mover a cobrinha pra cima, verificar se perdeu o jogo, etc.) e recebe como retorno uma estrutura do tipo Container (ver imagem abaixo) que contém um “snapshot” completo do jogo (maze + snake) e informações do jogo (maçãs, levels, etc.) Por ser um jogo baseado em blocos (cada posição só pode ser ocupada por uma Sprite) escolhi representar o “tabuleiro” em uma matriz (linhas e colunas) e a cobrinha é representada por uma lista.

Imagem não mais disponível

Numa versão anterior tentamos conteinizar a cobrinha junto com o tabuleiro em uma mesma matriz, mas isso resultou em problemas na renderização

Um passo interessante na renderização está em identificar qual textura cada Sprite que monta a cobrinha deve ter, mas creio que como este é um artigo apresentativo do jogo, este assunto fica para uma próxima postagem, mas é uma técnica interessante para se montar personagens que são formados por cadeias em um jogo, como uma cobra, um trem, até mesmo agrupamentos de objetos iguais, enfim.