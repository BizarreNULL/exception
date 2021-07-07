---
title: "Solucionando problemas de sincronismo de requisições em paralelo com SemaphoreSlim"
date: 2020-12-30T18:21:14-03:00
draft: false
author: "jojo"
tags: ["dotnet", "parallel", "async"]
---

# Dando um `push()` no problema

Um dos desafios que nem com as técnicas mais sombrias de *debugging* me ajudaram a entender o que meu contexto de requisições HTTP assíncronos e paralelos estavam retornando uma exceção baseada no cancelamento das referências das instâncias dos vários `CancellationToken` utilizados para construir o cliente das requisições.

Foi uma discussão bem legal com alguns monstros da CLR e até mesmo do Roslyn, que com uma paciência de Jó, conseguiram ir a fundo na explicação sobre o funcionamento da pilha de rede dentro do .NET, e também, sobre a abstração que estava consumindo. No geral, as diversas _tasks_ que havia emitido, que foram passadas para serem aguardadas sua execução até o fim como parâmetro da função `WhenAll` lá do binário `System.Runtime.dll`.

> Para não descer muito o nível, e deixar tudo de maneira bem entendível, não vou abranger muito a explicação que absorvi, e focar na resolução desse problema relativamente simples.



# Nem sempre é tudo uma questão de alto nível

Em contextos de paralelismo com tarefas que consomem muito tempo do processador, ainda que existem outras várias camadas para orquestrarem essas tarefas, desde de o próprio CPU até o sistema operacional e a própria *stack* de desenvolvimento, cabe a nós implementarmos controles eficazes na orquestração nas nossas próprias tarefas.

Especialmente nas linguagens de altíssimo nível, que entregam uma abstração *muito abstrata - eu ri por dentro aqui -* do que realmente é uma *thread*, devemos ter essa atenção dobrada para não cair em um contexto de cancelamento pois o tempo de resposta da aplicação para o ambiente onde foi delegada sua execução chegar ao limite.

Nesse cenário do `HttpClientHandler` e do `HttpClient` isso fica mais fácil de se perceber, mesmo que você implemente um *timeout* na casa dos `TimeSpam.FromDays(666)`, se o tempo de execução de qualquer uma das funções assíncronas do `HttpClient`, como a mais utilizada `HttpClient.SendAsync(HttpRequestMessage)`, podem retornar um *timeout*, mas não da resposta HTTP do servidor, e sim da sua `Task` que está no limbo do atual contexto de processamento.

> Vai por mim, evite utilizar funções que podem receber uma sobrecarga de uma instância de `CancellationToken`, se ela pode trabalhar com os *status* provenientes dessa estrutura, algum motivo tem.

Para simplificar o entendimento de todo esse cenário intangível, vamos trazer ao código uma péssima prática de programação (que quase me fez desistir do desenvolvimento do [HttpDoom](https://github.com/BizarreNULL/httpdoom/)) relacionada ao paralelismo e assincronicidade de código:

```csharp
using System;
using System.Linq;
using System.Net.Http;
using System.Threading.Tasks;

namespace Exception.Examples
{
    internal static class Program
    {
        public static async Task Main()
        {
             var tasks = await Task.WhenAll(Enumerable.Range(1, 5000)
                 .Select(_ => new HttpClient().GetAsync("https://google.com")));
             
             tasks
                 .Where(t => true)
                 .ToList()
                 .ForEach(r => Console.WriteLine($"Google answered {r.StatusCode}"));
        }
    }
}
```

> **Nota**: *Top-level programs* ainda tem algumas coisas bem peculiares para tratar inferência dinâmica de tipos e de contextos assíncronos e paralelos no C# 9, então vamos no clássico *entrypoint program* mesmo.

Um código bem simples, e que - se liga no *plot twist* - funciona perfeitamente bem! O motivo de tudo funcionar como deveria, é que temos implementado somente uma única responsabilidade dentro da função paralela e assíncrona, onde é iniciado 5000 `Task<HttpResponseMessage>` que será aguardada logo no seu *enclosure* que é justamente a própria `Task.WhenAll`, por ser uma expressão LINQ, o código pode ser um pouco difícil de ler, mas nada que alguns minutos de leitura não resolva.

O problema que estamos tentando criar, acontece quando além dessa única coisa que é feita (`return new HttpClient().GetAsync("https://google.com"))`, na expressão anônima da extensão `Select()` lá do LINQ), adicionamos alguma função que freia a execução dessa *thread*, por exemplo, escrever na tela informações da requisição conforme são executadas, vamos alterar um pouco o código e adicionar algumas funções que fação operações de IO:

```csharp
// ...
await Task.WhenAll(Enumerable.Range(1, 5000)
    .Select(async _ =>
    {
        try
        {
            var response = await new HttpClient().GetAsync("https://google.com");
            Console.Write($"Remote {response.RequestMessage?.RequestUri} answered {response.StatusCode}, ");

            var content = await response.Content.ReadAsByteArrayAsync();
            Console.WriteLine($"with a length of {content.Length} byte(s)");
        
            return response;
        }
        catch (System.Exception e)
        {
            Console.WriteLine(e.InnerException != null
              ? $"Error: {e.InnerException.Message}"
              : $"Error: {e.Message}");
        }
        
    }));
// ...
```

Nesse cenário como temos a interação com operações que causam problemas na hora de serem executadas, e nem por realizarem operações de IO com `response.Content.ReadAsByteArrayAsync()` mas sim pela escrita na tela usando `System.Console`, que causa um *lock* e vira uma bagunça, quem estiver disponível, executa essa função.

Quando você executar esse código (além de receber um belíssimo banimento da Google por realizar milhares de requisições em tão pouco tempo), vai se deparar com algumas exceções de `Task` ao invés das instâncias de classes referentes a pilha de rede do .NET, no caso a famosa mensagem "*The operation was canceled*".

E justamente quando receber essa resposta, é que os problemas do paralelismo começam a perturbar sua sanidade.

# Resolvendo com uma boa sinalização

Existem diversas APIs que são fornecidas dentro do CLR para sanar os problemas de paralelismo que correspondem a estes erros, podemos realizar a implementação mais simples que conheço, que tem pouquíssimas alterações no código fonte original da aplicação, utilizando `SemaphoreSlim` para corrigir esse fiasco.

Podemos resumir esse elemento de `System.Threads` como sendo uma entidade responsável por limitar o número de *threads* que podem acessar uma *pool* de recursos, que é justamente o que causa  "*The operation was canceled*" dentro de uma `Task`. Sua implementação é bem simples, e ele receber por padrão um único argumento correspondente a quantidade máxima de *threads* que podem ser alocadas para acessar estes recursos:

```c#
var semaphore = new SemaphoreSlim(Environment.ProcessorCount);
await Task.WhenAll(Enumerable.Range(1, 1000)
    .Select(async _ =>
    {
        await semaphore.WaitAsync();
        try
        {
            var response = await new HttpClient().GetAsync("https://zup.com.br");
            Console.Write($"Remote {response.RequestMessage?.RequestUri} answered {response.StatusCode}, ");

            var content = await response.Content.ReadAsByteArrayAsync();
            Console.WriteLine($"with a length of {content.Length} byte(s)");

            return response;
        }
        catch (System.Exception e)
        {
            Console.WriteLine(e.InnerException != null
                ? $"Error: {e.InnerException.Message}"
                : $"Error: {e.Message}");

            return null;
        }
        finally
        {
            semaphore.Release();
        }
    }));
```

Agora além de podermos controlar o número de *threads* que serão utilizados para realizar as requisições, também não vamos ter problemas de concorrência ao tentar processar essa *pool* de tarefas alocadas no fluxo de execução da aplicação.

