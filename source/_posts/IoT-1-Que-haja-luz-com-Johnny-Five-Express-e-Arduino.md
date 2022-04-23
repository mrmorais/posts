---
title: '[IoT 1] Que haja luz com Johnny-Five, Express e Arduino'
date: 2016-05-06 22:32:19
tags:
---

# [IoT 1] Que haja luz com Johnny-Five, Express e Arduino
06 de maio de 2016

[Ver código no GitHub][git]

Olá! Seja bem vindo ao meu GitHub blog. Eu tenho a intenção de escrever várias séries de "tutoriais" que possam ser colocados em prática e que, acima de tudo, sejam interessantes e explorem a criatividade para desenvolver mais coisas interessantes.

Neste post nós iremos criar uma aplicação que envolve Node JS, Express e Johnny-Five. Tudo isso controlando um LED numa placa Arduino (a minha é versão UNO, verifique no site do [Johnny-Five][jf] as placas compatíveis.)

### Express + Node JS
**Node JS**, por uma definição básica do [Wikipédia][wp-nodejs], é um interpretador de JavaScript que funciona no lado do servidor. É baseado no interpretador V8, da Google. Node JS é, sem dúvidas, algo muito poderoso e que possui muita força na comunidade, dá para fazer extremamente tudo com Node JS. Futuramente estaremos falando mais sobre ele e suas aplicações diversas.

Um dos framework para web desenvolvidos para Node JS é o **Express** (ou Express JS). É um frame bastante completo e que possui suporte para a maioria das views engines.

### Johnny-Five
Essa, assim como o Express, é mais uma biblioteca para NodeJS. **Johnny-Five** permite controlar placas como Arduino, BeagleBone, e muitas outras, com JavaScript através do NodeJS.

### Juntando as peças
Em resumo: NodeJS é muito rico em aplicações e principalmente é muito versátil. Podemos conectar duas bibliotecas inteiramente independentes e desenvolver soluções bastante criativas. E é isto que faremos hoje! Iremos utilizar o Johnny-Five para controlar a nossa placa Arduino e na mesma aplicação criar um servidor web para que esse controle possa ser feito através da rede ou, quem sabe, através da internet.

**Mãos ao código!!!**

(Certifique-se de possuir o NodeJS instalado)

A primeira coisa a se fazer é, em uma pasta vazia, instalar nosso pacote express e o johnny-five. No terminal:

    sudo npm install express --save
    sudo npm install johnny-five --save

Em outros post falaremos sobre o NPM e sobre como organizar um projeto com ele. Você pode criar uma aplicação Express através do comando "express nome-da-app", no entanto eu desencorajo esta prática e sugiro que você mesmo crie os arquivos da aplicação.

Crie o arquivo principal **index.js**:

    var express = require('express');
    var five = require('johnny-five');
    
Indica ao NodeJS que a aplicação requer express e johnny-five instalados
    
    var board = new five.Board();
    var led;
    var status = "Apagado";
    
    board.on("ready", function() {
    	led = new five.Led(13);
    });
    
A construção de `five.Board()` faz uma varredura nas portas seriais do computador na finalidade de encontrar uma placa a ser controlada. A variável status irá servir apenas para exibir ou alterar o estado do LED futuramente, por enquanto é Apagado.

`board.on()` é executado quando a placa foi encontrada e está pronta para rodar. Neste estágio nós inicializamos um LED na porta 13 da placa.
    
    var app = express();
    app.use(express.static(__dirname + '/public'));
    
Usaremos a variável app para gerenciar a nossa instancia de uma aplicação Express. A função `use()`, neste caso, o diretório de arquivos estáticos do nosso site. Como normalmente em qualquer site. Aqui foi escolhido a pasta /public.
 
    
    app.post('/led', function(req, res) {
    	var command = req.body.acao;
    	console.log(command);
    	if (command == "apagar") {
    		console.log("Apagando...");
    		if(led.off()){
    			status = "Apagado";
    			console.log("Sucesso. Apagado.");
    
    			res.send(status);
    		} else {
    			res.send("error");
    		}
    	}
    
    	if (command == "acender") {
    		console.log("Acendendo...");
    		if(led.on()){
    			status = "Aceso";
    			console.log("Sucesso. Aceso.");
    
    			res.send(status);
    		} else {
    			res.send("error");
    		}
    	}
    });

`app.post()` ou `app.get()` serão usadas para responder às requisições HTTP do respectivo tipo, POST ou GET (também existem outros). Este com o POST receberá requisições por AJAX com um parametro 'acao'. Dependendo da ação recebida ele irá executar o comando `led.on()`, para acender, ou `led.off()`, para apagar. E a resposta (`res.send()`) será o novo estado do LED.
    
    app.get('/', function(req, res) {
    	res.render('home', {ledStatus: status});
    });
    
Essa resposta GET irá renderizar a página inicial e passar o estado do led.
    
    
    app.listen(3000, function() {
    	console.log('Express started');
    });
    
Por fim, o aplicativo é colocado para "rodar" na porta 3000.

Bem, agora temos o cerne de nossa aplicação. Agora precisamos criar uma view. A view irá incorporar o jQuery que será utilizado para enviar requisições por ajax para POST '/led' que irá tratar ações. Não irei detalhar como a view foi feita, sugiro que baixe o código-fonte deste tutorial para ver como ficou na íntegra. 

[Ver código no GitHub][git]

![](http://i.imgur.com/q5jEcTe.png)

Para rodar o código é muito simples. No terminal:

    node index.js

Antes de executar certifique-se de preparar sua placa com um led no pino 13 e ligar em uma porta USB. 

![](http://johnny-five.io/img/breadboard/led-13.png)

Algo interessante para se fazer é testar isso pelo smartphone conectado à rede. Para isso é necessário liberar a porta 3000 do seu computador na interface de rede:

    iptables -A INPUT -p tcp -i wlan0 --dport 3000 -j ACCEPT
    
Espero ter contribuído com este tutorial! Em breve trarei mais coisas interessantes.

[Ver código no GitHub][git]

   [jf]: <http://johnny-five.io>
   [wp-nodejs]: <https://pt.wikipedia.org/wiki/Node.js>
   [git]: <https://github.com/mrmorais/tutoriais/tree/master/johnny-five-led>
