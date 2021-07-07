---
title: "Hackeando apps financeiros para OSINT (e pela zueira): Tudo pelo PIX!"
date: 2021-01-10T16:47:21-03:00
draft: false
author: "jojo"
tags: ["android-re", "research"]
---

O PIX foi uma das maiores mudanças que tivemos promovidas pelo BACEN (e por outras instituições financeiras ligadas aos fóruns), que vai promover muita agilidade, simplicidade, e o mais importante segurança. Um dos pontos mais perigosos de vender um produto, ou uma *feature* dele como sendo algo seguro, é que muitas vezes isso pode ser utilizado contra você.

Para não focarmos no PIX, e ir direto ao assunto, deixo como recomendação uma postagem feita pelo mestre Anchises lá no *blog do posto Ipiranga*, você pode encontra-lá [aqui](https://anchisesbr.blogspot.com/2020/10/seguranca-aspectos-de-seguranca-do-pix.html), recomendo que faça a leitura do conteúdo, antes de avançar por aqui.

Antes de cairmos em cima do banco, ou *fintech* que escolhi para efetuar o *bypassing* de todas as funcionalidades de segurança da aplicação - ofuscação de payloads não é bala de prata, uma dica importante - vou trazer um trecho do próprio Anchises da sua postagem em seu blog:

> [Segundo o Banco Central](https://www.bcb.gov.br/estabilidadefinanceira/pagamentosinstantaneos), o objetivo é oferecer pagamento instantâneo seja tão fácil, simples, intuitiva e rápida quanto realizar um pagamento com dinheiro em espécie. Os pagadores poderão iniciar pagamentos usando chaves ou apelidos para a identificação da conta transacional como o número do telefone celular, o CPF, o CNPJ ou um endereço de e-mail (que devem ser cadastrados no seu banco), por meio de QR Code (estático ou dinâmico) ou com uso da tecnologia NFC (near-field communication) para pagamentos por aproximação.

É importante notarmos esses itens relacionados a como a chave pode ser formada, bem como as informações que são contidas nela. Sua chave PIX, pode ser seu **telefone celular**, **CPF**, **CNPJ**, **endereço de e-mail** ou uma chave gerada "aleatoriamente" e vinculada a sua conta.

Quando alguém realiza uma transferência de valores utilizando o PIX, temos um `whois` de quem está por de trás daquela chave, trazendo informações como **nome completo**, **CPF mascarado** (na verdade alguns bancos esqueceram dessa parte para clientes da mesma instituição, como sempre, o OSINT agradece, mas acredito que não deveria funcionar dessa forma 👀, ou muitas vezes vazando o número do documento consultando o comprovante de pagamento, que não trazia esse dado para o cliente *mobile*) e o **banco do destinatário**.

Alguns cenários que podemos mapear fazendo uma análise dessa superfície de ataque, são relativamente pequenos, como encontrar o nome completo de uma pessoa que cadastrou um e-mail como sua chave. Porém para dentro da cena de OSINT, esse é um recurso muito poderoso, especialmente pelas tratativas das famosas pessoas publicamente expostas, já que agora é possível chegar no banco que um político ou milionário guarda sua grana (e também seus *offshores*, caso for uma instituição nacional).

Outro ponto muito interessante que podemos explorar, além de enumerar massivamente chaves PIX, enriquecendo cada vez mais uma lista de e-mails, ou de *phishing*, é utilizar-se da função de transferir R$ 0,01 centavos (durante minhas análises, algumas *fintechs* me permitiram inclusive adicionar dinheiro que não existia, tipos fortes são seus amigos, não deixe passar `float`, `double` e `decimal` como a mesma coisa em sua *stack*), com o bónus de conter uma mensagem com **qualquer conteúdo**, e bom, isso é perigoso em diversas formas.

> Isso inclusive virou meme, e graças a esse meme das mensagens cornas via PIX, que surgiu a maliciosidade nos olhos da galera, para montar campanhas de *marketing* totalmente direcionadas, pela bagatela de um centavo a mensagem.

# O escolhido (depois de uma longa análise de outros candidatos)

Depois de olhar alguns dos principais meios de pagamento que tinham implementado o PIX, acabei por selecionar uma aplicação Android muito bem escrita para utilizar nos testes (principalmente pois já tinha uma conta aberta lá, que nunca utilizei), foi bom para relembrar muita coisa das magias ocultas que um bom framework de instrumentação pode fazer, nesse caso utilizei o vovô XPOSED, e depois acabei migrando para o Frida já que não precisava trocar nada na ROM padrão do AVD.

Essa aplicação em especial, me chamou bastante atenção na forma que foi implementado os recursos de segurança, mas um pequeno deslize por parte dos desenvolvedores, permitiram que todas as suas funcionalidades fossem facilmente (na verdade eu levei 5 horas olhando SMALI e umas chamadas malucas do JNI) ignoradas por um famoso `try {} catch {}` implementado de forma genérica.

Como já dizia o ditado, de nada adianta pagar milhões para GuardSquare para deixar seus executáveis devidamente ofuscados, sendo que o "feijão com arroz" do *clean code* não é colocado quando a esteira de desenvolvimento ferve. Na verdade encontrei esse mesmo detalhe em outras aplicações nacionais.

Nessa mesma aplicação, notei também a falta de sanitização dessa mensagem do PIX, e acabei por escrever um fuzzer para tentar pegar alguma coisa na renderização da mensagem, talvez traga esse item em outra postagem. Enfim, continuando a análise, percebi que o *bypass* rolou, graças a forma que essas funções de segurança eram implementadas: Havia uma *activity* principal, que era executada antes de todas as outras (no começo parecia uma gambiarra, depois parecia que estava no começo), e quando era feito esse *handling* dos estados das telas da aplicação, quando essa principal era instanciada, as verificações de segurança, eram realizadas.

E devido ao `try {} catch {}` do código que tentava carregar essa tela de aviso (sabe aquela famosa "*por motivos de segurança seu dispositivo Android blá blá blá*"), era pulada uma das exceções possíveis, que era justamente o *handle* para a entidade que fazia referência para essa *activity* na memória ser nula, caiu para dentro do limbo das exceções genéricas, e o código continua sua vida depois daquilo, então foi literalmente algo similar a isso para contornar esses controles de segurança:

```javascript
var randomSecurity2 = Java.use('???');

randomSecurity2.???.implementation = function () {
    console.log("a ???() called, returning nothing.");
}

randomSecurity2.???.implementation = function () {
    console.log("a ???() called, returning nothing.");
}

// ??? quer dizer que escondi os nomes para evitar problemas.
```

Demorei bastante para chegar aqui, devido a ofuscação de código do DexGuard, quebrar os decompiladores clássicos dos executáveis Dalvik, colocando caracteres unicode no meio do nome da função, classe, propriedade etc, fazendo que o decompilador, ao tentar montar o pseudocódigo Java, à partir do SMALI, caia em um *loop* de exceções. Ainda bem que podemos criar padrões de *hooking* baseados nas assinaturas de tipo das funções, e também enumerar uma classe por essas mesmas características em seu construtor.

À partir daqui, foi só programar uma extensão para meu proxy para conseguir enviar as requisições para o servidor de *backend* com a mesma chave que o cliente original utilizaria para criptografar as *payloads* enviadas. Depois de todo esse trabalho, comecei a automatizar as requisições para conseguir, finalmente, fazer o *fingerprint* de chaves PIX, e também zuar alguns amigos com centenas de transações de um centavo com frases aleatórias do Chuck Norris (caso um dia tiver um domingo tedioso, recomendo utilizar [essa API](https://api.chucknorris.io/) para azucrinar uns amigos). 



# ShellScript e `jq` para começar a automação

Depois de fazer todas as muambas necessárias para fazer o *script* funcionar, temos essa belezura em ação:

![PIXMAP funcinando c:](https://i.imgur.com/YdWoOtm.png)

E agora era só passar a chave de alguém, e ser feliz:

```sh
➜  automator ./pixmap.sh --key="+5511......."
[+] Key +5511......., of type PHONE intel:
 >  ISPB: 18236120
 >  Institution: NU PAGAMENTOS S.A.
 >  Owner: Jonas Uliana
 >  Owner Type: PERSONAL
 >  Document: 000........

```

Só substitui meus dados por pontos, assim que alguns bancos responderem adequadamente sobre alguns pontos que notei de bem perigoso na arvore de riscos ao analisar o fluxo transacional do PIX, irei também deixar o *script*, e uma ferramenta de OSINT (isso é OSINT? Não sei dizer muito bem) decente de fato para azucrinar seus amigos e enriquecer a lista de phishing da empresa.



# Minhas conclusões

Ainda está se popularizando, cada vez mais com maior adoções e popularidade, mas te todos os testes que fiz em diferentes meios de pagamentos (incluindo bancos, *fintechs* e tudo mais), é visível que existe muita falta de maturidade dos modelos de risco ao lidar com fluxos fraudulentos ou anómalos dentro do PIX.

Outro ponto que não sei dizer se é um risco, ou uma regra de negócio à ser seguida do BACEN, é a permissão de uma conta poder pesquisar centenas de milhares de outras chaves PIX de uma só vez. Notei que nenhuma instituição tinha implementado controles de *rate limiting* ou similares, diretamente em suas APIs. A propósito, foi bem legal notar diversas dessas empresas utilizando GraphQL, eu particularmente prefiro bastante, e acredito que qualquer um que desenvolva *frontend* compartilhe desse mesmo sentimento.

E sobre bloquear o aceite de mensagens no PIX? Bom, acredito que isso pode ser implementado no meio de pagamento, não quero receber mensagens de ameaça/marketing ou nada do tipo, ainda que isso implique em ganhar alguns centavos. Meus amigos que caíram na minha zueira, fizeram 10 reais cada, mas para totalizar essa grana, tomaram 1000 transações PIX com frases do Chuck Norris.