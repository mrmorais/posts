---
title: Scraprice - API de monitoramento de preços
date: 2016-05-21 22:40:37
tags:
---

# Scraprice - API de monitoramento de preços
21 de maio de 2016

Na postagem anterior apresentei um módulo NPM que faz WebScraping com produtos da loja Submarino e "converte" o link do produto em informações do produto. 

Utilizando este módulo que desenvolvi, criei um site com Node, Angular e Express usando um banco de dados MongoDB e eis o resultado: Scraprice. Esta ferramenta faz algo bastante simples: além de fazer o scraping e armazenar os dados, ele diariamente roda um script que busca o preço do produto e coloca em uma array de todos os preços já computados. Nada diferente do que é feito, por exemplo, no Zoom ou Buscapé.

Por enquanto a API ainda não está documentada e o código ainda não está no GitHub.

[Acessar o Scraprice][sp]

A aplicação está hospedada com [Open Shift / Red Hat Cloud][rhc], com 1GB de espaço. E o banco de dados está no [MLab][mlab], 500MB.

![](http://i.imgur.com/UUynpp0.png)

    
[subscrap]:<http://mrmorais.github.io/2016/05/17/web-scraping-com-produtos-da-submarino-com.html>
[rhc]:<https://openshift.redhat.com>
[mlab]:<http://mlab.com/>
[sp]:<http://scraprice-mrmorais.rhcloud.com/>
