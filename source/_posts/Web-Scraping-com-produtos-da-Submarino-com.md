---
title: Web Scraping com produtos da Submarino.com
date: 2016-05-17 22:39:09
tags:
---

# Web Scraping com produtos da Submarino.com
17 de maio de 2016

![Logomarca da Submarino](http://1.bp.blogspot.com/-uO60O4C0BTQ/Ue88sjiS3MI/AAAAAAABK2o/baYvdCguuu8/s1600/submarino+logo.jpg)

Olá! Desenvolvi, hoje, um algoritmo muito simplista que tem como funcionalidade realizar Web Scraping em uma página de qualquer produto da empresa Submarino e retornar os atributos daquele produto como objeto.

Web Scraping é uma técnica bastante simples e muito utilizada pelos robôs da Google, que buscam e qualificam conteúdos na internet para serem buscados pelo navegador. Esta técnica consiste em ler todo o HTML de uma determinada página, como se fosse interpretá-la. Parecido com o que fazem os leitores de tela.

Pois bem, com Web Scraping é possível realizar a obtenção de dados de qualquer tipo de página e com produtos esta regra torna-se bem mais simples já que sempre existe uma pré-disposição dos elementos de uma página, o que nós chamamos de _layout_. Portanto, o meu algoritmo sabe qual o _layout_ das paginas de produtos da Submarino e, com isso, ele consegue ler as informações de quase todos os produtos.

Eu acabei de disponibilizar o pacote desta simples aplicação (ao abrir, você verá que é realmente simples) e está disponível no NPM com o nome **submarino-scrap** ([acessar][npm])

#### Por quê?

A aplicação deste simples algoritmo pode ser útil em diversas coisas. A primeira coisa que deve ser notada é que nenhuma empresa possui uma API pública de seus produtos, mesmo que estas informações possam ser de interesse para diversas aplicações.

Fazer uma API pública destas informações permitiria o desenvolvimento de diversas aplicações legais (e abertas) que façam por exemplo a monitoria da variação dos preços dos produtos (ver como funciona a "Black Friday"), base estatística, mineração de dados, etc.

A ideia é fazer e disponibilizar um web scraping para cada grande e-commerce da internet e por fim criar um pacote que integre todos os demais e que faça a leitura de produtos da maioria dos sites de venda do Brasil.

Não irei fazer tutorial deste pacote aqui no sie, uma exemplo prático já se encontra na [página do NPM][npm] e você pode conferir lá. Nos próximos dias estarei criando os novos "scraps" e também disponibilizando um repositório no GitHub para que mais pessoas possam se tornar contribuidoras.

[npm]:<https://www.npmjs.com/package/submarino-scrap>
