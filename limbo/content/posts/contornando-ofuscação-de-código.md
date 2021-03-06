---
title: "Contornando a ofuscação de código no Android"
date: 2021-03-05T23:32:22-03:00
draft: true
tags: ["android-re", "research"]
---

Das diversas coisas que já fiz em segurança de aplicações móveis, e dessas, principalmente testes totalmente *blackbox* (aqueles onde você tem o aplicativo, `.apk`, e deve procurar por vulnerabilidades nesse escopo incerto), acabou por ficar sendo uma das atividades que mais gosto de exercer no *bug hunting* e também como lazer. 

> **Alerta**: O texto é recheado de alguns RANTs, mas acredito que são necessários. 

Antes de aprofundar sobre o tema, gostaria de deixar um *disclaimer*: **Se você trabalha, ou se intitula, como *pentester* ou correlatos, focados em segurança em aplicações móveis, no mínimo você deve saber programar muito bem na *stack* que está auditando**, ou rabiscar *Hello, World!* de acordo com a sua senioridade. No geral, considero que, se caso você estiver em uma enrascada para analisar uma aplicação, e não souber desenvolver em nada naquela plataforma, ou se quer entende de fato o que acontece por de trás das cortinas, com certeza está *passando o carro na frente dos bois*.

Entender ao menos o básico do que está fazendo, é crucial para conseguir chegar do outro lado. Aposto com você, que o *Orange-Tsai* não indexou tantas vulnerabilidades críticas em aplicações baseadas na JVM (e também em PHP, e algumas outras dezenas de *stacks* de desenvolvimento) sem conhecer exatamente o que acontece até mesmo além da linguagem. Dito isso, talvez iniciar uma trilha de estudos focada nos conceitos básicos de desenvolvimento Android (para iOS, acredito que devo escrever algo focado especificamente para essa plataforma, e trazer um desenvolvedor nativo com mais experiência, não me sinto confortável para falar de Apple, visto que não atuo com seus produtos no dia-a-dia), e também de segurança ofensiva para ela.

> Na parte de *dev*, recomendo assinar a [Alura](https://www.alura.com.br/), e estudar um pouco de nativo, e também de aplicações híbridas. Claro que o ideal, é focar seus estudos na *stack* que mais atua. No meu caso, fico no Flutter, Xamarin e no NDK, que também já entraria em uma postagem exclusiva para ele (mas dando um *spoiler*, você não precisa do Android Studio para desenvolver para Android). Já na parte de segurança ofensiva, o melhor conteúdo em nosso idioma, é feito pelo grande [Maycon Vitali](https://maycon.hacknroll.io/br/about/), com a [Hack N' Roll Academy](https://www.youtube.com/channel/UCcYYP7JizTd24W9Mr7FIhxw).



# A Sua Assinatura

Começando com uma boa indicação de trilha sonora para o texto que vem a seguir, também vamos rever alguns entendimentos sobre o que é ofuscação de código, e o motivo da segurança por obscuridade - se é que a ofuscação de código pode ser considerada isso, eu sinceramente não sei responder, mas acredito que não. Mas acabo por ficar meio na dúvida, pois a vantagem, assim como na maioria das vezes, fica do lado do atacante -, ser tão amplamente implementada em aplicações Android. Em especial, aquelas de cunho financeiro, como *fintechs* e bancos no Brasil, na verdade já notei que diversas vezes os times de *Application Security* levam esse assunto e *certificate pinning* como bala de prata para aumentar a maturidade de segurança de uma aplicação, tentando deixar mais difícil a vida de quem vai audita-lá no futuro.

> A final de contas, para que utilizar a [Safetynet](https://developer.android.com/training/safetynet/attestation), se você pode reter informações críticas de negócio no *frontend* ou até mesmo deixar algum segredo por lá, temos a ofuscação de código à caminho! (Acredito que a [Guardsquare](https://www.guardsquare.com/pt-br) deve abrir um sorriso no rosto com essa mentalidade).

E sobre o tema anterior, voltamos a focar no item **o que é ofuscação de código**. Afinal de contas, para a maioria das consultorias de segurança, análise de segurança ofensiva em aplicações Android, se limita em realizar o *bypass* de alguma coisa, e focar nas interações com a internet (AKA olhar somente para RESTful ou algum outro SDK que faça chamadas para a internet) ao invés da aplicação.

*Short long answer*, com as soluções já consagradas de mercado, como o [{Pro, Dex}Guard](https://www.guardsquare.com/en/blog/dexguard-vs-proguard), seus contextos de aplicação vão bem além de somente ofuscar um código. Até na variante *open-source*, o ProGuard, ele efetua a otimização do *bytecode*, remove trechos de códigos que não podem ser acessíveis devido a algum fluxo lógico na aplicação, e de fato, efetua a ofuscação baseado em um dicionário.

Na versão *enterprise*, o DexGuard, o negócio fica mais tenso. A proteção vai bem além de somente um *bytecode* direcionado para a JVM, e sai desse contexto que estamos comentando de análise estática, e vai até nas etapas de instrumentação durante as análises dinâmicas, como por exemplo, realizar o *hooking* de alguma função com o [Frida](https://github.com/frida/frida). Acho que a parte mais legal, é de fato ofuscar até as chamadas criptográficas sem gerar (muitos) problemas no código, e também chegando ao âmbito de bibliotecas de código nativo, como alguns bancos nacionais o Facebook e diversos outros aplicativos delegam boa parte da suas funcionalidades de segurança e também do *core business* da aplicação.

Mas aqui vem um ponto muito importante, o ProGuard foca mais na miniturialização do código, adicionando funções **muito simples e primitivas** de ofuscação de código. Acredito que a única coisa que realmente seja ofuscada dentro da utilização do ProGuard, fique na troca das assinaturas dos nomes de classes, propriedades e funções. E que na maioria das vezes, o dicionário utilizado é bem determinístico, além da existência de diversas ferramentas de *deobfuscation* feitas especialmente para ele (um *paper* já antigo, mas que gostei muito quando li, escrito por um trio com artigos muito bem escritos, é o [Anti-ProGuard: Towards Automated Deobfuscation of Android Apps](https://www.researchgate.net/publication/320542477_Anti-ProGuard_Towards_Automated_Deobfuscation_of_Android_Apps), além do próprio [Reconstruct](https://github.com/LXGaming/Reconstruct), mas que claramente, é dependente do *mapping* dessas substituições) e também outros ofuscadores de código, acredito que os projetos mais mantidos são o [Simplify](https://github.com/CalebFenton/simplify) e o [dex-oracle](https://github.com/CalebFenton/dex-oracle), que foi a inspiração e base para várias outros *deobfuscators*.

No fim, nada vai substituir a sua capacidade de interpretar e entender código intermediário e pseudocódigo oriundo de algum decompilador. Um bom exercício é justamente estender as funcionalidade de decompiladores *open-source* ou que expõem alguma API de desenvolvimento, para olhar justamente para os padrões de códigos das funções que são interessantes para sua análise. Para não perder o costume, entenda bem da linguagem de programação utilizada, bem como o ambiente onde ela funcionará. 

---

> **Nota**: Esse artigo ainda é um *draft*, por tanto não pode remeter ao que de fato será escrito no final, se é que um dia vou terminar de escrever.

 

