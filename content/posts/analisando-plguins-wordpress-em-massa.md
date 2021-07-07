---
title: "Analisando plguins wordpress em massa"
date: 2021-07-06T23:38:32-03:00
draft: true
tags: ["scraping", "research"]
author = "jojo"
_build:
    list: never
---

Acredito que a grande maioria do pessoal que procura vulnerabilidades rentáveis dentro do mundo de aplicações web, em algum momento já passou pela *stack* que o WordPress fornece, seja auditando seu `core`, ou plugins, sendo este último o foco dessa postagem.

Junto a um simples *crawler* escrito em C# 9.0, vamos percorrer todos os *plugins* listáveis na *index* do site do projeto, e organiza-lo em uma estrutura de descrição, reputação, número de downloads, e o próprio `.zip` contendo o código do plugin, isso tudo organizado em diretórios separados por nomes, e um arquivo `.json` de metadados do pacote.

> **Nota**: Caso tenha interesse em apenas utilizar o *crawler*, verifique-o no repositório indicado no item *Conclusão*.

# Top-level program

Por não haver a necessidade de termos uma estrutura de programa complexa, vamos utilizar os recursos que o C# 9.0 tem de melhor, em trazer minimalismo na hora de construir uma aplicação de linha de comando: Com *top-level programs*. Dentro dessa linguagem, essa caracteristica permite que escvrevamos código, sem o *template* clássico do `static void Main()`.

![Apresentação do site](/static/26f475e9f2a847fe0bbf2eab.png)
