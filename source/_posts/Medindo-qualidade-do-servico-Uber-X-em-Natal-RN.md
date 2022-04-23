---
title: Medindo qualidade do serviço Uber X em Natal/RN
date: 2017-09-23
tags: []
---

# Medindo qualidade do serviço Uber X em Natal/RN
23 de setembro de 2017

Em março 2016 o The Washington Post lançou uma [matéria super interessante](https://www.washingtonpost.com/news/wonk/wp/2016/03/10/uber-seems-to-offer-better-service-in-areas-with-more-white-people-that-raises-some-tough-questions/?utm_term=.21414a6a4db8) da jornalista computacional Jennifer Stark e do professor Nicholas Diakopoulos. A reportagem indagava que o Uber em Los Angeles aparenta ter um serviço melhor em áreas com mais pessoas brancas, e para chegar a esta conclusão eles consumiram durante um mês a API pública do Uber e mediram a qualidade do serviço em todos os bairros.

A ideia é bastante simples: foi utilizado como indicador de qualidade o tempo (estimado pelo aplicativo) que um motorista levaria para chegar até o local em que o cliente se encontra, esta informação pode ser obtida pela [API do Uber](https://developer.uber.com/docs/riders/ride-requests/tutorials/api/python). Então, basta mapear diversos pontos distribuidos por toda cidade e periodicamente armazenar os tempos estimados. Depois é só analizar os dados obtidos com ferramentas adequadas.

Este foi o resultado do estudo do The Washington Post:

![](https://img.washingtonpost.com/wp-apps/imrs.php?src=https://img.washingtonpost.com/blogs/wonkblog/files/2016/03/uber1.png&w=1484)

Em 26 de agosto deste ano (2017) fez um ano desde que o Uber passou a operar em Natal/RN. Motivados pela [notícia compartilhada pelo professor Fernando Masanori](https://www.facebook.com/fmasanori/posts/10214024510345174), eu e meu colega Daniel Marx decidimos explorar um pouco da API Uber e reproduzir o que Jennifer e Nicholas fizeram com os dados do Uber em Los Angeles.

## Mapeando a cidade

A primeira informação importante para este estudo é as localizações e bairros que deverão ter as estimativas de espera rastreadas. Então, mapeamos diversos pontos na cidade de Natal, um total de 86 localidades – Natal tem 36 bairros, cada bairro teve entre 2 e 3 localidades mapeadas dependendo do tamanho do bairro.

Por engano, o bairro Nova Parnamirim não foi mapeado

Este é o mapa de localidades construído:

![](https://github.com/mrmorais/uber-map-natal/blob/master/imgs/pontos_mapeados.png?raw=true)

As coordenadas (latitude e longitude) dos pontos mapeados foram armazenadas em uma estrutura de dados para leitura pelo script de requisições. Observe que os pontos foram escolhidos aleatoriamente – o trabalho do The Washington Post faz uma seleção uniforme dos pontos (ou seja, todos pontos têm uma mesma distancia dos pontos adjacentes).

## Executando as requisições

Foi criado um script em Python que a cada 3 minutos faz 86 requisições (uma para cada localidade) à API do Uber, e salva os tempos estimados retornados pela API em um arquivo, para analise posterior. Esse script ficou em execução durante 5 dias (entre 14 e 19 de setembro) tendo efetuado quase 157 mil requisições.

## Resultado

O tratamento posterior dos dados que fizemos foi efetuar uma média, por bairro, do tempo de estimativa de espera. Por fim, representamos as médias obtidas em um mapa e chegamos a este resultado:

![](https://github.com/mrmorais/uber-map-natal/blob/master/imgs/without_lbls.png?raw=true)

Todos os códigos utilizados neste estudo estão disponíveis no meu Github [neste endereço](https://github.com/mrmorais/uber-map-natal).