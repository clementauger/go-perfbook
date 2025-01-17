# Escrevendo e otimizando o código Go

Este documento descreve boas práticas para escrever código Go de alto desempenho.

Embora existam discussões focadas em tornar os serviços individuais mais rápidos (cache, etc), projetar sistemas distribuídos de alto desempenho vai além do escopo deste trabalho. Já existem bons textos sobre monitoramento e projeto de sistemas distribuídos. Eles englobam um conjunto totalmente diferente de decisões de pesquisa e design.

Todo o conteúdo será licenciado sob o CC-BY-SA.

Este livro está dividido em diferentes seções:

1. Dicas básicas para escrever software que não é lento
    * Material de nível introdutório (CS 101)
1. Dicas para escrever software rápido
    * Veja seções específicas sobre como obter o melhor do Go
1. Dicas avançadas para escrever software *realmente* rápido
    * Para quando o seu código otimizado não é rápido o suficiente

Podemos resumir estas três seções como:

1. "Seja razoável"
1. "Seja intencional"
1. "Seja perigoso"

## Quando e onde otimizar

Estou colocando isso em primeiro lugar porque é realmente o passo mais importante. Você deveria mesmo estar fazendo isso?

Toda otimização tem um custo. Geralmente esse custo é expresso em termos de complexidade de código ou carga cognitiva - o código otimizado é raramente mais simples do que a versão sem otimizações.

Mas há outro lado que chamarei de economia da otimização. Como programador, seu tempo é valioso. Há o custo da oportunidade de trabalhar em outras coisas no projeto, por exemplo, quais erros corrigir ou quais funcionalidades adicionar. Otimizar as coisas é divertido, mas nem sempre é a tarefa certa a escolher. O desempenho é uma característica (*feature*), mas entrega e corretude também são.

Escolha a coisa mais importante para trabalhar. Às vezes não é em uma otimização da CPU, mas na experiência do usuário. Algo tão simples como adicionar uma barra de progresso, ou tornar uma página mais responsiva ao fazer cálculos no plano de fundo depois de renderizar a página.

Às vezes isso será óbvio: um relatório de hora em hora que leva três horas para ficar pronto é, provavelmente, menos útil do que aquele que é concluído em menos de uma hora.

Só porque algo é fácil de otimizar não significa que vale a pena ser otimizado. Ignorar os casos mais fáceis é uma estratégia de desenvolvimento válida.

Pense nisso como uma otimização do *seu* tempo.

Você pode escolher o que otimizar e quando otimizar. Você pode mover o controle deslizante entre "Software rápido" e "Implantação rápida".

As pessoas ouvem e repetem "a otimização prematura é a raiz de todo mal", mas eles perdem o contexto completo da citação.

"Os programadores gastam muito tempo pensando ou se preocupando com a velocidade de partes não-críticas de seus programas. Estas tentativas de conseguir eficiência tem um forte impacto negativo quando a depuração e manutenção são consideradas. Devemos esquecer pequenas eficiências, digamos cerca de 97% do tempo: a otimização prematura é a raiz de todo o mal. Porém, não devemos deixar passar nossas oportunidades nesses 3% críticos."
-- Knuth (*tradução livre*)

Adicione: https://www.youtube.com/watch?time_continue=429&v=RT46MpK39rQ
   * não ignore as otimizações fáceis
   * mais conhecimento de algoritmos e estruturas de dados torna mais otimizações "fáceis" ou "óbvias"

"Você deve otimizar? Sim, mas somente se o problema for importante, o programa é realmente muito lento e há alguma expectativa de que pode ser feito mais rápido, mantendo a corretude, robustez e clareza".
-- A prática da programação, Kernighan e Pike (*tradução livre*)

[BitFunnel performance estimation] (http://bitfunnel.org/strangeloop) tem alguns números que tornam esta decisão mais explícita. Imagine uma máquina de busca hipotética que precisa de 30.000 máquinas em vários *datacenters*. Essas máquinas tem um custo de aproximadamente US$ 1.000 por ano. Se você pode dobrar a velocidade do software, isso pode economizar US$ 15 milhões por ano. Até mesmo um único desenvolvedor gastando um ano inteiro para melhorar o desempenho em apenas 1% irá se pagar.

Na grande maioria dos casos, o tamanho e a velocidade de um programa não são uma preocupação. A otimização mais fácil é não ter que fazê-la. A segunda otimização mais fácil está apenas comprando hardware mais rápido.

Uma vez decidido que você irá mudar seu programa, continue lendo.

## Como otimizar

### Fluxo de otimização


Antes de entrarmos nos detalhes, vamos falar sobre o processo geral de otimização.

Otimização é uma forma de refatoração. Entretanto, em vez de melhorar algum aspecto do código-fonte (duplicação de código, clareza, etc), ela melhora algum aspecto de desempenho como, por exemplo, reduzir o uso da CPU, reduzir a ocupação da memória, reduzir a latência, etc. Essas melhorias, geralmente, são implementadas a troco de alguma perda na legibilidade do código. Isso significa que além de um conjunto de testes unitários (para garantir que suas mudanças não irão quebrar nada), você também precisará de um bom conjunto de _benchmarks_ para garantir que suas mudanças estão, de fato, entregando o ganho de desempenho desejado. Você deve ser capaz de verificar se a alteração realmente está reduzindo o uso da CPU. Às vezes uma alteração que você pensava que iria melhorar o desempenho, na verdade não gera nenhum impacto ou, até, causa um impacto negativo. Nesses casos, sempre desfaça suas alterações.

<cite>[Qual é o melhor comentário que você já encontrou em um código? - Stack Overflow](https://stackoverflow.com/questions/184618/what-is-the-best-comment-in-source-code-you-have-ever-encountered)</cite>:
<pre>
//
// Caro mantenedor:
//
// Assim que desistir de tentar "otimizar" essa rotina,
// e perceber que terrível engano você cometeu,
// por favor, incremente o contador a seguir como uma forma de aviso
// à próxima pessoa:
//
// total_hours_wasted_here = 42
//
</pre>

Os _benchmarks_ que você decidir usar devem ser precisos e devem oferecer números reproduzíveis em cargas relevantes. Se execuções individuais tiverem uma variância muito alta, isso tornará mais difícil a detecção de pequenas melhorias. Assim, você precisará usar o [benchstat](https://golang.org/x/perf/benchstat) ou uma solução equivalente para realizar testes estatísticos já que não conseguirá verificar as melhorias apenas via observação. (Note que a utilização de testes estatísticos é uma boa ideia em qualquer cenário). Os passos para executar os _benchmarks_ devem estar documentados e quaisquer scripts e/ou ferramentas adicionais devem ser incluídas no repositório com instruções de como utilizá-los. Esteja atento a grandes conjuntos de _benchmark_ que requerem muito tempo para sua execução: isso irá tornar o processo de desenvolvimento mais lento.

Lembre-se, também, que tudo que pode ser medido pode ser otimizado. Tenha certeza de que está medindo a coisa certa.

O próximo passo é decidir qual é o seu objetivo com a otimização. Se o objetivo é melhorar o uso da CPU, qual velocidade é aceitável? Você quer melhorar o desempenho em 2x ou em 10x? Você pode definir isso como "um problema grande como N que precisa ser resolvido num tempo menor que T"? Você está tentando reduzir o uso de memória? Em quanto? Para uma determinada redução de uso de memória, quão mais lento é aceitável? Do que você está disposto a abrir mão em troca de menos exigência de espaço?

Otimização com foco em latência de serviços é uma proposta mais complicada. Livros inteiros foram escritos sobre como testar o desempenho de servidores web. A principal questão é que, para uma única função, o desempenho é bastante consistente para um problema de determinado tamanho. Para _web services_, você não tem um único número. Um bom conjunto de _benchmark_ para _web services_ fornecerá uma distribuição de latência para um dado nível de requisições por segundo. Esta palestra dá uma boa visão geral de alguns dos problemas:["How NOT to Measure Latency" by Gil Tene](https://youtu.be/lJ8ydIuPFeU)

TODO: Veja a seção a seguir sobre otimização de _web services_.

As metas de desempenho devem ser específicas. (Quase) sempre você será capaz de fazer algo ser mais rápido. Otimização é, frequentemente, um jogo de retornos decrescentes. Você precisa saber a hora de parar. Quanto esforço a mais você vai fazer para obter aquela pequena melhora? Quão disposto você está a fazer um código mais feio e mais difícil de manter?

A palestra de Dan Luu mencionada anteriormente em [BitFunnel performance estimation](http://bitfunnel.org/strangeloop) apresenta um exemplo do uso de cálculo aproximados para determinar se as metas de desempenho estimadas são razoáveis.

TODO: Programming Pearls tem "Problemas de Fermi". Conhecer os slides de Jeff Dean ajuda.

Para o desenvolvimento de novos projetos, você não deve deixar o a avaliação de desempenho para o fim. É fácil dizer "depois eu faço", mas se o desempenho é realmente importante, isso deve ser considerado desde a concepção do projeto. Quaisquer alterações relevantes na arquitetura para consertar problemas de desempenho serão ainda mais arriscadas quando tiverem que ser feitas próximas do prazo final. Perceba que, *durante* o desenvolvimento, o foco deve ser um desenho coerente, algoritmos e estrutura de dados. Otimização em níveis mais baixos da estrutura devem aguardar uma fase mais avançada do ciclo de desenvolvimento, quando houver uma visão mais completa do desempenho do sistema. Qualquer perfil completo de sistema que você faz enquanto o sistema está incompleto oferecerá uma visão distorcida de onde os gargalos estarão, de fato, no sistema acabado.

TODO: Como evitar/detectar "Morte por mil cortes (Lingchi)" por _software_ mal escrito.

O _benchmarking_ como parte do CI é difícil devido a interferências causadas pelo compartilhamento de recursos. Difícil, também, de ser ativado em métricas de desempenho. Um bom meio termo é ter _benchmarks_ executados pelo desenvolvedor (em hardware apropriado) e incluídos nos _commits_ que abordam, especificamente, o desempenho. Para aqueles que são apenas patches gerais, tente identificar as potenciais degradações de desempenho "a olho nu", na revisão de código.

TODO: Como acompanhar o desemepnho ao longo do tempo?

Escreva código que você pode comparar. Você pode fazer perfilamento em sistemas maiores, porém em _benchmarking_ você quer testar partes isoladas. Você precisa ser capaz de extrair e configurar o contexto necessário para que os _benchmarks_ executem testes representativos e suficientes.

A lacuna entre sua meta e o desempenho atual também te darão uma orientação de por onde começar. Se você precisa de apenas 10% a 20% de melhoria de desempenho, provavelmente você consegue alcançar isso com pequenos ajustes. Se você precisa de uma melhoria da ordem de 10x ou mais, isso vai exigir mudanças maiores em sua estrutura.

Um bom trabalho em aprimoramento de desempenho exige conhecimentos dos mais variados níveis, desde desenho de sistemas, rede, _hardware_ (CPU, caches, armazenamento), algoritmos, ajustes e _debugging_. Com tempo e recursos limitados, considere aquele que lhe dará o maior ganho: nem sempre será o algoritmo ou um ajuste fino no programa.

Em geral, as otimizações devem ocorrer de cima para baixo. Otimizações em nível de sistema terão mais impacto que aquelas em nível de código. Certifique-se de que você está resolvendo o problema no nível apropriado.

Esse livro irá tratar, em sua maior parte, sobre redução de uso da CPU, redução de uso da memória e redução de latência. É interessante destacar que, raramente, você fará os três ao mesmo tempo. Talvez o tempo de CPU esteja mais rápido, mas agora seu programa usa mais memória. Talvez você precise reduzir o espaço de memória, mas agora o programa levará mais tempo.

[Lei de Amdahl](https://en.wikipedia.org/wiki/Amdahl%27s_law) diz para nos concentrarmos nos gargalos. Se você dobra a velocidade da rotina que toma 5% do tempo de execução, houve um ganho de apenas 2,5% no tempo total. Por outro lado, aumentar a velocidade da rotina que toma 80% do tempo em apenas 10% oferece um ganho real de 8%. Perfilamento irá ajudar a identificar onde o tempo é realmente gasto.

Quando se está otimizando, você quer reduzir o trabalho que a CPU precisa fazer. _Quicksort_ é mais rápido que _bubble sort_ porque resolve o mesmo problema em menos passos. É um algoritmo mais eficiente. Você reduziu o trabalho que a CPU tem para executar a mesma tarefa.

O ajuste do programa, como as otimizações do compilador, geralmente trazem uma pequena melhora no tempo total de execução. Grandes vitórias quase sempre vêm de uma mudança algorítmica ou uma mudança na estrutura de dados ou uma mudança fundamental na forma como o seu
programa é organizado. A tecnologia de compiladores melhora, mas lentamente. A [Lei de Proebsting](http://proebsting.cs.arizona.edu/law.html) diz que compiladores melhoram seu desempenho em 2x a cada 18 *anos*, um contraste gritante com a Lei de Moore que diz que o desempenho dos processadores dobra a cada 18 *meses*. Melhorias em algoritmos funcionam em magnitudes maiores. Algoritmos para programação inteira mista [melhoraram por um fator de 30.000 entre 1991 e 2008](https://agtb.wordpress.com/2010/12/23/progress-in-algorithms-beats-moore%E2%80%99s-law/). Para um exemplo mais concreto, considere [essa decisão](https://medium.com/@buckhx/unwinding-uber-s-most-efficient-service-406413c5871d) de substituir de um algoritmo geo-espacial de força bruta descrito em um post do blog do Uber por um mais especializado e mais adequado para a tarefa apresentada. Não há mudança de compilador que lhe dará um aumento equivalente no desempenho.

Um perfilador pode mostrar que muito tempo é gasto em uma rotina específica. Pode ser que esta seja uma rotina cara ou uma rotina barata
porém que é chamada muitas vezes. Em vez de imediatamente tentar aumentar a velocidade de uma rotina, veja se você pode reduzir o número de vezes que ela é chamada ou, até mesmo, eliminá-la completamente. Vamos discutir estratégias de otimização mais concretas na próxima seção.

As três perguntas da otimização:

* Nós precisamos fazer isso mesmo? O código mais rápido é aquele nunca executado.
* Se sim, esse é o melhor algoritmo?
* Se sim, essa é a melhor *implementação* desse algoritmo?
