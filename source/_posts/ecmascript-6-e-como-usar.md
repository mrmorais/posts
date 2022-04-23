---
title: ECMAScript 6 e como usar
date: 2017-03-23 22:15:20
tags: [ES6, Node.JS]
---

# ECMAScript 6 e como usar
23 de março de 2017

No [artigo anterior](/Aplicacao-real-time-com-socket-io-pt-1/) comentei que a aplicação que estamos desenvolvendo depende de um ambiente que deve ser setado para que todos os módulos sejam importados e que os interpretadores entendam o código que estamos passando.

## ECMAScript (?)
Esse nome pode parecer estranho a primeira vista. Oras, nossa aplicação é em JavaScript, o que seria então esse ECMAScript?

[ECMAScript](https://pt.wikipedia.org/wiki/ECMAScript) é uma linguagem de programação na qual o JavaScript é baseado, como também o ActionScript. Atualmente, existem dois padrões bem difundidos do ECMAScript, o 5 (ou ES5, ou ECMAScript 5) que é o padrão que por _default_ todos os browsers conhecidos do mercado aceitam, e o ES6 que não tem, ainda, todas suas funcionalidades aceitas pelos browsers.

O ES6 é o padrão para o futuro do ECMAScript, começou a ser desenvolvido em 2011 e começou a ser difundido em 2015 [[1](https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Suporte_ao_ECMAScript_6_na_Mozilla)]. O ES6 nada mais é que o ES5 melhorado, todas as coisas que o ES5 pode fazer, o ES6 também pode, mas a recíproca não é válida.

Os principais browsers já dão suporte à vários recursos do ES6 ([neste link](https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Suporte_ao_ECMAScript_6_na_Mozilla) você pode ver quais recursos o Firefox já suporta do ES6). O fato é que nem todos os browsers já têm esse suporte, e para isso existem ferramentas (transcripters) que basicamente pega nosso código ES6 e o transcreve para ES5, que é suportado por todos os navegadores.

Ai que entra o [Babel](http://babeljs.io/), um conhecido transcripter para ES6, com ele você irá programar com o JavaScript do futuro mesmo que os navegadores ainda não aceitem.

Note que quando eu falo navegador eu estou me referindo, na verdade, à qualquer interpretador de código JS. Isso faz com que esta regra (de suporte ao ES6) também seja válida para o Node.JS. Como sabemos (ou não), o Node utiliza a engine de interpretação do Chrome, o V8. Em outras palavras: se o Google Chrome ainda não suporta completamente o ES6, o Node.JS também não o faz.

O ES6 possui diversas novas _features_. Tentarei abordar algumas delea, principalmente as mais utilizadas, mas você pode acessar, [nesse link](http://es6-features.org), uma referência que contém todas as especificações das novas _features_ do ES6, e utilizá-lo como base de consulta.

Irei mostrar como fazer a instalação do Babel e otimizar a "transcrição" com NPM, montando um cenário ideal para rodar o nosso projeto [Pig Web Dice](http://goo.gl/8FrA5d). E em seguida veremos algumas novas _features_ do ES6.

## Montando um set up Babel
Para utilizar o Babel, você precisará de um conhecimento básico sobre NPM (o gerenciador de pacotes para Node.js), ele já vem instalado com o Node. Verifique com um `npm -v` a versão do npm, o Babel utiliza alguns "sub-pacotes" em comum entre os que veremos a seguir e utilizar o npm versão 2.x pode causar problemas de performance devido à forma pela qual o npm 2.x importa os pacotes. Portanto, faça o upgrade para a versão 3.x do npm.

O babel possui diferentes pacotes, mas estes serão os necessários para usar com ES6:
- `babel-core`: Principal pacote do Babel.
- `babel-cli`: Necessário para executar comandos a partir do `package.json` ou do Terminal.
- `babel-preset-env`: Preset do babel para determinar os plugins necessários com base nos ambientes que você deseja suportar o ES6.

Vamos lá! crie o diretório para a sua aplicação, considerarei que se chama `ecmascript-6` (`mkdir ecmascript-6`) e entre no diretório (`cd ecmascript-6`). O primeiro passo é inicializar o diretório com um arquivo de configuração do `npm`, para isso execute:
```
npm init -y
```
O argumento `-y` diz para o init que não queremos utilizar o modo iterativo: inserir as informações linha por linha. Perceba que o npm criou um arquivo chamado `package.json` na raíz do diretório. Esse arquivo possui todas as informações sobre nossa aplicação, tais como: nome, versão, dependencias e scripts. Inicialmente ele deve ser algo como isso:

**package.json**
```json
{
  "name": "ecmascript-6",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

Primeiro vamos fazer algumas modificações básicas, como inserir o nome do autor, dar uma descrição, etc. O meu ficou assim:

**package.json**
```json
{
  "name": "ecmascript-6",
  "version": "1.0.0",
  "description": "Boilerplate para Node.JS com ES6",
  "main": "index.js",
  "scripts": {
  },
  "keywords": ["es6", "babel", "node"],
  "author": "Maradona Morais",
  "license": "ISC"
}
```
Perfeito. Agora que temos o `package.json`, o npm sabe como instalar nossas dependencias. Então vamos dizer ao npm que queremos instalar os pacotes anteriormente mencionados do babel:
```
npm i --save-dev babel-core babel-cli babel-preset-env
```
O `i` é uma forma curta do comando `install`. O `--save-dev` diz para o npm que as dependencias que estamos instalando são necessárias apenas para o ambiente de desenvolvimento, não são importantes para rodar aplicação em si. Perceba que na raiz foi criado o diretório `node_modules`, é aí que ficam todos os códigos dos pacotes instalados pelo npm.

Aqui preciso abrir um parênteses para explicar como funciona o reconhecimento de importação no Node. Quando fazemos a importação de um módulo no código Node, fazemos algo como `var express = require('express');`, o node faz a busca pelo módulo `express` em dois lugares: no `node_modules` local (o que está na raiz do diretório) e, se não encontrar aí, procura no `node_modules` global (que são os módulos instalados no seu computador, não no diretório). Para instalar um módulo globalmente faça `npm i -g nome-do-pacote`. Fechando parênteses.

O arquivo `package.json` foi alterado pelo npm, e agora deve está assim:

**package.json**
```json
{
  "name": "ecmascript-6",
  "version": "1.0.0",
  "description": "Boilerplate para Node.JS com ES6",
  "main": "index.js",
  "scripts": {},
  "keywords": [
    "es6",
    "babel",
    "node"
  ],
  "author": "Maradona Morais",
  "license": "ISC",
  "devDependencies": {
    "babel-cli": "^6.24.0",
    "babel-core": "^6.24.0",
    "babel-preset-env": "^1.2.2"
  }
}
```
Ok. O Babel está instalado com sucesso. Mas ainda precisamos configurar o babel, para isso devemos criar um arquivo na raíz do projeto chamado `.babelrc`, com o seguinte conteúdo:

**.babelrc**
```json
{
  "presets": ["env"]
}
```
Esse simples arquivo diz para o babel qual o(s) preset(s) que ele deve utilizar ao fazer o _transcript_, aqui estamos utilizando apenas o [env](https://babeljs.io/docs/plugins/preset-env/) que, como dito anteriormente, determina quais plugins necessários.

Já estamos quase aptos para rodar um código com o ES6, falta apenas dizer como o babel irá rodar esse código. Vamos criar alguns scripts npm no arquivo `package.json`, adicione isso nas chaves do valor "scripts":

**package.json**
```json
"scripts": {
  "start": "babel-node src/index.js",
  "build": "babel src -d dist",
  "serve": "node dist/index.js"
}
```
Estes três scripts fazem o seguinte:
- `start`: Inicia a aplicação com o transcript temporário. Com o `babel-node` o babel vai, basicamente, transcrever o código de `src/index.js` de ES6 para ES5, e guardar o novo código em algum folder "tmp". Quando a aplicação terminar de rodar, ou quando você der um comando para parar o processo, o código transcriptado é perdido.
- `build`: Aqui o código é transcriptado mas não executado. O babel vai "converter" todo código que está no diretório `src` para ES6 e salvar no diretório `dist`.
- `serve`: Com esse script simplesmente mandamos rodar a aplicação que está no `dist`, essa aplicação já está em ES5, então o comando `node` entende todo o código sem ajuda do babel.

Crie os diretórios `src` e `dist` na raíz do seu projeto, e dentro de `src` crie o arquivo `index.js`, com o conteúdo:
```JavaScript
function f(x, ...y) {
  return x * y.length;
}
f(3, "hello", true) == 6
```
A definição `...y` diz que, além o `x`, nossa função `f` também pode receber mais de 1 argumento, que é passado para a função como vetor. Na chamada, passamos o `3` para `x`; o `"hello"` para `y[0]` e `true` para `y[1]`. Essa é uma _feature_ do ES6. [ver mais sobre isso](https://babeljs.io/learn-es2015/#ecmascript-2015-features-default-rest-spread).

Teste executar `npm start`, `npm run build` e `npm run serve` para verificar se está tudo ocorrendo como esperado, se sim agora você tem um ambiente pronto para desenvolver Node.JS com ES6.

Os diretório do projeto, como de costume, está disponível no GitHub e você pode acessar por [este link](https://github.com/mrmorais/tutoriais).

## Novas _features_ do ES
As novas funcionalidades que vêm com o ECMAScript 6 você pode ter acesso por [esta referência](es6-features.org/) e também pelo próprio babel [neste link](https://babeljs.io/learn-es2015/). Os exemplos mostrados abaixo, e neste artigo como todo, foram retirados do babeljs.io.

### Arrows functions
O ES6 introduz um novo conceito de funções, as Arrows functios (função seta?), denotada pela expressão `(arg)=>{code}`. Além da sintaxe ser diferente, o Arrow function já vem com o `bind` do `this` apontado para o bloco superior de códigos. Esse tipo de função é mais utilizado como `callback` (se não sabe o que é callback entenda como uma função que é passada como parâmetro de outra função). Veja um exemplo:
```javascript
nums.forEach(v => {
  if (v % 5 === 0)
    fives.push(v);
});
```
A arrow function nesse exemplo está sendo passada para o `forEach` do array `nums`, e recebe como parâmetro apenas um argumento (`v`), veja que os parênteses não são obrigatórios quando temos apenas um argumento. Dentro das `{}` temos um bloco de códigos convencional.

### Let e Const
No lugar do famoso `var` para inicializar uma variável, o ES6 introduz o `let`. E o `const` é utilizado para criar uma "variável" de assinatura única, ou seja, que não tem seu valor alterado em tempo de execução. Exemplos:
```JavaScript
var x = 10; // ES5
let x = 10; // ES6

const nome = "babel";
nome = "node"; //Erro, const não pode ter o valor alterado
```

Enfim. Existem diversas novas funcionalidades no ES6 e recomendo que você aprenda este novo padrão, pois é o futuro do JavaScript.

Espero que este artigo lhe tenha sido útil. Vejo você no próximo. Lembre-se que os códigos deste artigo estão [neste repositório do GitHub](https://github.com/mrmorais/tutoriais).
