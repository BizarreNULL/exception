---
title: "Analisando plguins wordpress em massa"
date: 2021-07-06T23:38:32-03:00
draft: true
tags: ["scraping", "research"]
author: "jojo"
_build:
    list: never
---

Acredito que a grande maioria do pessoal que procura vulnerabilidades rentáveis dentro do mundo de aplicações web, em algum momento já passou pela *stack* que o WordPress fornece, seja auditando seu `core`, ou plugins, sendo este último o foco dessa postagem.

Junto a um simples *crawler* escrito em C# 9.0, vamos percorrer todos os *plugins* listáveis na *index* do site do projeto, e organiza-lo em uma estrutura de descrição, reputação, número de downloads, e o próprio `.zip` contendo o código do plugin, isso tudo organizado em diretórios separados por nomes, e um arquivo `.json` de metadados do pacote.

> **Nota**: Caso tenha interesse em apenas utilizar o *crawler*, verifique-o no repositório indicado no item *Conclusão*.

# Top-level program

Por não haver a necessidade de termos uma estrutura de programa complexa, vamos utilizar os recursos que o C# 9.0 tem de melhor, em trazer minimalismo na hora de construir uma aplicação de linha de comando: Com *top-level programs*. Dentro dessa linguagem, essa caracteristica permite que escvrevamos código, sem o *template* clássico do `static void Main()`.

![Apresentação do site](/static/26f475e9f2a847fe0bbf2eab.png)

Para análisar o HTML da página, e não passar raiva ou perder tempo com expressões regulares ou ficar refém de algo como `LastIndexOf()` ou `IndexOf()`, irei utilizar o `AngleParse` como biblioteca para realizar o *parsing* da sopa de letras que vamos extrair do site exibido acima. Um ponto de atenção na hora de fazer o *crawling*, é justamente se antentar onde será realizado, tendo em vista que a listagem completa dos quase 60 mil plugins não é possível.

> **Nota**: Também vamos fazer tudo isso assíncrono e paralelo, utilizando `async` e `SemaphoreSlim`.

Ah, e seguindo a linha das outras postagens, o código vou escrevendo conforme vou conseguindo tempo no meu dia-a-dia, por tanto essa postagem também se remete a essa condição, e pode ser atualizada constantemente conforme meu humor varie. Caso tenha visto alguma coisa que alterou, todo o blog está disponível em meu GitHub no `/exception`.

```csharp
using System;
using System.Net.Http;
using System.Threading;
using static System.Console;

var clientHandler = new HttpClientHandler
{
    AllowAutoRedirect = false,
    UseCookies = false
};
var client = new HttpClient(clientHandler);
var semaphore = new SemaphoreSlim(Environment.ProcessorCount);
var downloadList = new[]
{
    "https://wordpress.org/plugins/browse/popular/",
    "https://wordpress.org/plugins/browse/beta/",
    "https://wordpress.org/plugins/browse/featured/",
    "https://wordpress.org/plugins/browse/blocks/"
};

WriteLine("[+] Crawler started with a possible total of {0} thread(s)", Environment.ProcessorCount);

```
