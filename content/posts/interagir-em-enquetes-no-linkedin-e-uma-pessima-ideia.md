+++
title = "Interagir em enquetes abertas no LinkedIn é uma péssima ideia!"
date = "2021-03-14T15:31:10-03:00"
author = "jojo"
authorTwitter = "6a6f6a6f"
cover = ""
tags = ["scraping", "csharp"]
keywords = ["sharp", "data"]
showFullContent = false

+++

Diversas vezes me deparo na rede social *voltada para trabalho* (uma nota pessoal sobre, politica nacional, tem **muito** haver com o nosso trabalho), inumeras enquetes no estilo interaja com determinada ação, para exibir qual a sua opção dentre as possíveis enumeradas em determinada postagem, que são um prato cheio para o pessoal que gosta de analisar perfils, ou de inteligência. Atualmente, isso vem se popularizando muito, por conta de cada vez mais atritos entre o Bolsonaro, com o retorno oficial de Lula à cena deplorável da politica brasileira.

> **Nota**: Não gosto de ambos os possíveis presidenciáveis de 2022, mas assumo que repudio o descaso do atual presidente a cena que vivemos na pandemia, bem como a falta de transparência do presidente rival em entregar mais clareza dos fatos pessoais de seus julgamentos. Mas pela várzea que vivemos, acredito que não preciso  explicar minha opinião para essas duas figuras.

Antes de mais nada, acredito que qualquer fluxo de tratamento de dados que possa ser especializado sem muito esforço por uma parte, sempre vai ser uma péssima ação, qualquer tipo de interação com essas entidades. Afinal de contas, a rede da Microsoft já tem acesso a todos os nossos padrões de comportamento, e a única ressalva que temos é as politicas e leis de proteção aos dados pessoais (que parece valer menos caso você for um utilizador chinês na plataforma, ou não for um cidadão de mesma nacionalidade da plataforma, mas esses casos de conflitos de interesses entre ocidente e oriente, bem como os próprios interesses do LinkedIn, fica para outra postagem).

Como o LinkedIn é uma rede que tende a ser aberta para quem tem vincúlo com a pessoa `X` ou `Y`, não esperem de mim censurar os nomes e perfils, bem como o resultado dessa análise. Para ficar com o de costume, caso encontrar seus dados aqui na minha análise, e precisar esconde-los daqui, basta abrir uma *issue* ou uma *pull request* no blog, através de seu repositório no GitHub.

# Enfim coisas técnicas

Fui para o caminho mais rápido possível para isso, poderia ter analisado o aplicativo ou tentado encontrar algo já pronto para essa finalidade, mas acredito que a diversão de trabalhar com escopos de análise de dados em plataformas que não dispõe com clareza formas de interagir com os dados ali presentes, um desafio simples, mas muito produtivo no aspecto de mapear perfils em redes sociais, seja ela do tipo que for.

Como de praste, minha *stack* de desenvolvimento vai ser o C# 9, junto do .NET5 (em breve pretendo mudar o *target* para quando a *preview* 3 do .NET6 estiver disponível, especialmente para brincar no Apple Silicon, e também explorar mais a MAUI, além de não ter problemas com a RFC do *float*). Para o *parsing* da sopinha de *tags* do HTML, vou tentar uma approach o mais *vanilla* possível, utilizando `IndexOf()`, `Remove()` e funções nativas da linguagem, para garantir uma performance consistente, sem adicionar bibliotecas complexas para uma finalidade muito simples.

### Analisando a requisição de uma postagem

A primeira coisa que vou fazer (estou escrevendo enquanto vou descobrindo a melhor forma de fazer isso acontecer, então caso queira apenas o resultado final, vá direto ao repositório referênciado no final da postagem) é analisar a aba *network* do navegador, ao interagir com uma postagem:

```shell
curl 'https://www.linkedin.com/voyager/api/feed/reactions?count=10&q=reactionType&start=70&threadUrn=urn:li:activity:XXXXX614171169XXXXX' \
  -H '...' \
  --compressed
```

Realmente esperei que seria algo mais complexo, mas o LinkedIn facilita bastante, já que sua API RESTful fornece o *endpoint* `reactions` de fácil acesso a qualquer usuário, claro que no cabeçalho da requisiçao, existia minha sessão, que por motivos óbvios, acabei removendo-a para evitar problemas com os amigos *h4x05s*.

Além disso, os parâmetros de paginação, `start` e `count` vão facilitar bastante na hora de realizar as requisições de fato! Da forma facilitada que estamos vendo, acredito que uma alternativa 100% válida seria utilizar somente ShellScript junto do `jq`, acabaria por resolver esse primeiro ponto. Mas como quero fazer algo com um pouco mais de funções, vou prosseguir com a ideia do C#, e pelo visto, nem será necessário quebrar a cabeça com o *parsing* de qualquer trecho da página.

Expondo o *dataset* que vou consumir, será [essa postagem](https://www.linkedin.com/posts/wanderson-santos-136a1bb0_e-a%C3%AD-o-que-ser%C3%A1-de-n%C3%B3s-activity-6775961417116983296-2aR0/).

Com tantas informações já disponíveis, vamos codificar um pouco! Graças ao `record` do C# 9, abstrair esses modelos de dados ficaram muito simples, e extremamente performáticos:

![A foto do progresso!](/static/image-20210314164216249.png)

Evoluindo para um melhor entendimento da resposta vinda da API, que não é tão fácil de entender a primeiro momento, mas depois fica bem tranquilo de entender (existe formas bem mais ficientes de ser implementado o *parsing*):

![Enumerando usuários paginados na requisição](/static/image-20210314174239109.png)

Depois de continuar evoluindo o código para respeitar (parcialmente) a paginação, e entendendo melhor como funciona o comportamento dos servidores do LinkedIn, em especial, sobre *rate limiting*, foi só uma questão de escrever um código XGH para extrair todo o conteúdo, e indexa-lo em um arquivo `.csv` para uma análise mais profunda:

![O *dataset* formado](/static/image-20210314200041144.png)

Para simpliificar, e não fazer diversas requisições ao *backend* do LinkedIn, escolhi aleatóriamente 1064 pessoas daquela postagem. Por fim o código em C# ficou uma bagunça, mas cumpriu seu propósito de forma majestosa, indexando todos esses resultados em apenas alguns segundos - tudo isso pode ser parametrizado através dos parâmetros da própria API da rede social.

Um ponto de atenção, é sobre os *cookies* necessários para estar autenticado, e também, autorizado dentro do LinkedIn:

```csharp
var session = new CookieContainer();

session.Add(new Cookie("JSESSIONID", "...", "/", ".www.linkedin.com"));
session.Add(new Cookie("li_at", "...", "/", ".www.linkedin.com"));
```

Todos os demais *cookies* são apenas *trackers* ou caso você utilize um provedor de SSO. Para os *headers*, para minha surpresa, só é necessário o seguinte:

```csharp
var reactionsMessage = new HttpRequestMessage(HttpMethod.Get, post)
{
    Headers =
    {
        {"csrf-token", "..."}
    }
};
```

E claro, configurar adequadamente seu `HttpClient` e `HttpClientHandler`. Deixei os *cookies* apontados para o objeto `session`, do primeiro *snippet* de código, dessa forma ficou tudo bem organizado da seguinte forma:

```csharp
using var clientHandler = new HttpClientHandler
{
    UseCookies = true,
    AllowAutoRedirect = false,
    CookieContainer = session
};

using var client = new HttpClient(clientHandler);
```

Depois de tudo isso, veio uma parte meia feia, no que se diz respeito as melhores práticas para nomear os tipos dentro do C#, isso acontece nessa versão do .NET5 por conta da falta de suporte da *annotation* `JsonPropertyName`, de ser utilizada em `record` e também em seus construtores, então para simplificar o código, abstrai apenas os elementos que foram importantes para essa análise:

```csharp
private record Paging(int start, int total, int count);
private record description(string text);
private record name(string text);
private record Element(string reactionType, description description, name name);
```

De certa forma ficou simples, porém dava para ter feito melhor. Com as limitações da `System.Text.Json`, acredito que por enquanto, esse seria o mais minificado que conseguiria chegar por enquanto. Para realizar o *parsing* do JSON que é respondido, fiz algo bem simples - mas sentindo muita falta da simplicidade de tratar tipos de forma dinâmica, igual temos na NewtonSoftware, mas como sempre prefiro fazer sem utilizar nada além da *std*, acho que está de bom tamanho:

```csharp
var content = await response.Content.ReadAsStringAsync();
var payload = JsonSerializer.Deserialize<JsonElement>(content);

var paging = JsonSerializer.Deserialize<Paging>(payload.GetProperty("paging").GetRawText());
if (paging == null) return;
```

E dessa forma seguimos o *parsing* para os demais `record`:

```c#
var elements = JsonSerializer.Deserialize<List<Element>>(payload.GetProperty("elements").GetRawText());
```

Na verdade todos eles estão referenciados explicitamente pelo `Element`, dessa forma, basta iterar sobre a lista e colher os dados que julgar necessário. No meu caso imprimi na tela, conforme nas primeiras imagens, para ter uma noção de como as coisas estavam funcionando:

```c#
elements.ForEach(e =>
{
    if (e == null) return;

    if (e.name != null && !string.IsNullOrEmpty(e.name.text))
    {
        Console.WriteLine($" > User: {e.name.text.ToUpperInvariant()}");
    }
    if (e.description != null && !string.IsNullOrEmpty(e.description.text))
    {
        Console.WriteLine($" > Role: {e.description.text.ToUpperInvariant()}");
    }
    if (e.reactionType != null && !string.IsNullOrEmpty(e.reactionType))
    {
        Console.WriteLine($" > Type: {e.reactionType}");
    }
});
```

Ainda que não temos algo dinâmico, as referênciais internas dos `record` ainda podem ser traiçoeiras se acessadas diretamente, como tentar navegar diretamente para `e.name.text`, que pode resultar em uma referência nula dentro do escopo de execução, por isso temos também a validação de `e.name != null`.

Depois disso tudo, só utilizo um `Distinct()` e indexo todo o resultado em um `.csv`:

```csharp
await File.WriteAllLinesAsync("result.csv", textToWrite.Distinct().ToList());
```

E tudo isso vai ter como produto, o que vamos utilizar para analisar de forma mais profunda os dados, de quem gosta mais de `X` ou `Y`.

# Quem é quem?

Vamos analisar brevemente os dados produzidos. Para simplificar, realizei a conversão desse `.csv` para um `.sqlite` (poderia ter utilizado algo mais simples com `pandas`) e consumindo os resultados pela própria *shell* do `sqlite3`:

```sqlite
sqlite> .schema
CREATE TABLE CRAW(
  "NAME" TEXT,
  "ROLE" TEXT,
  "REACTION" TEXT
);
sqlite> SELECT COUNT() FROM CRAW;
1064
```

Para a postagem que utilizei como referência, temos:

- À favor do Bolsonaro: Quem possivelmente teve sua reação como sendo `LIKE`;
- À favor do Lula: Quem possívelmente teve sua reação como sendo `EMPATHY`.

Com isso em pauta, vamos olhar a quantidade de pessoas que votaram em cada uma das figuras políticas:

```sqlite
sqlite> SELECT COUNT() FROM CRAW WHERE REACTION = "EMPATHY";
319
sqlite> SELECT COUNT() FROM CRAW WHERE REACTION = "LIKE";
668
```

Nesse *dataset* que colhi da postagem, chegamos ao total de 319 votos à favor do Lula, e 668 votos para o Bolsonaro. Porém, da mesma forma que podemos apenas realizar um `COUNT()`, também podemos olhar apenas para os nomes de quem possívelmente prefere governo `X` ou `Y`, começando pelo candidato de direita:

```sqlite
sqlite> SELECT NAME FROM CRAW WHERE REACTION = "LIKE" LIMIT 10;
NADINE JUNKES
MATHEUS CHAGAS
BRUNO JARED CRUZ
JOÃO GANDOLFI
URIEL OLIVEIRA BEZERRA
ADONAI DUTRA
CARLOS EVERTON DOS SANTOS GERALDO
JEFFERSON J.
SARA ELAINE LOPES PEREIRA
MÁRCIO ROSA
```

E por fim, Lula:

```sqlite
sqlite> SELECT NAME FROM CRAW WHERE REACTION = "EMPATHY" LIMIT 10;
ANA CECÍLIA MACHADO
GIOVANNA DUARTE ALMEIDA
LUCAS CATTA PRÊTA
SHEILA PATRICIO
DANIEL TOMAZELLI RAMOS
CRISTIANO FERREIRA
SUELLEN CAROLINE
SABRINA BECKER
VINICIUS T.
VICTÓRIA LAPENNA RODRIGUES
```

Também podemos analisar os cargos de cada um, e correlaciona-los com o possível ideal político base de dada um. Da mesma forma, podemos utilizar essa mesma análise para determinar o cunho político de uma empresa, marca ou qualquer página no LinkedIn que fique atrelada a uma pessoa.

# Conclusão

Em poucos minutos (na verdade foi um intervalo de algumas horas, domingo é dia de arrumar a casa, e meus gatos conseguiram se superar essa semana), vimos a possibilidade de escrever um *crawler* muito simples para análisar uma postagem relacionada a uma *poll* baseada em reações, e chegar em cada um dos perfils que reagiram individualmente.

Lembre-se que fiz isso sem buscar qualquer resultado ou benefício disso, mas sabemos que existem centenas de empresas que fazem diferente. Da mesma forma, imagine a quantidade de empresas que possívelmente te barrou em um processo seletivo, por analisar rapidamente seu perfil com alguma dessas ferramentas, e verificou que seu ideal político, ou seja lá qual for os dados que a empresa estiver analisando, não está de acordo com ela.

No final do dia, todos nós somos dados, números e métricas. Dentro do LinkedIn, especialmente pelo fato de existir uma concepção ou idéia de que todos somos autênticos (no sentido de não ter uma grande concentração de perfils de identidade incertas, ou *fakes* de outras pessoas), isso se torna **muito** mais perigoso. Então tenha mais cuidado ao interagir com esse tipo de informação, seus dados podem estar nesse momento passando na mão da empresa que você acabou de aplicar.

> **Nota**: Ao ver que ainda existem pessoas que carregam políticos de estimação, vejo que 2022 vai ser um ano muito complicado. Também aproveito o momento para repetir: Por mim ambas as figuras presidenciáveis são, no mínimo, ridículas.