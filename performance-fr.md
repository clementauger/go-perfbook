# Ecrire et optimiser du code Go

Ce document fournit les pratiques et méthodes pour écrire du code haute-performance avec le langage de programmation Go.

Bien que ce document aborde les manières d'améliorer l'efficacité d'exécution
de services (cache, etc), la conception de système distribués performant ne sera
pas traité. Il existe déjà de la litterature de qualités au sujet du monitoring
et de la conception de systèmes distribués. Optimisation de tels systèmes distribués
nécessite des choix et des recherches totalement différentes de celles discutées
dans ce document.

Ce contenu utilise la license CC-BY-SA.

Ce livre est partagé en plusieurs sections:

1. Techniques de base pour écrire pour ne pas écrire du "code-lent"
    * Les bases de l'optimisation
2. Techniques de base pour écrire du code performant
    * Particularités de la programmation Go
3. Techniques avancées pour écrire du code *vraiment* rapide
    * Lorsque l'optimisation de code n'est plus suffisante

Nous pouvons résumer ces trois sections ainsi:

1. "être raisonnable"
2. "être intelligent"
3. "être téméraire"

## Où et quand optimiser

J'évoque ce sujet en premier car c'est vraiment l'étape la plus importante
de tout processus d'optimisation. Devriez vous vraiment dépenser votre énergie
dans cette action ?

Chaque optimisation à un coût. Généralement ce coût est exprimé en termes
de complexité ou de charge cognitive -- un code optimisé est rarement plus
simple que son alter non optimisé.

Mais il y à une autre manière d'aborder cette question que je nomme
le "coût économique" de l'optimisation. En tant que programmeur, votre temps
est coûteux. En cela le temps que vous dépensez dans ces optimisations ne pourra
pas être dépensés pour d'autres activités tels que la correction de bugs, l'ajout
de nouvelles fonctionnalités. Optimiser le code est une activité satisfaisante,
cependant ce n'est pas toujours le bon objectif à poursuivre. La performance est
une fonctionnalité, au même titre que la livraison, au même titre que l'exactitude.

Choisissez avec soins ce qui est le plus important à améliorer. Parfois ce n'est
pas l'efficacité de la consommation du CPU, mais l'expérience utilisateur. Cela
peut être des choses aussi simple que l'ajout d'une bar de progression, ou de
transformer le chargement d'une page en implémentant un design responsive en
réalisant les calculs en arrière plan ou en post-rendering.

Parfois ce sera évident: un rapport horaire qui s'exécute en trois heures
est très certainement moins utile que le même rapport qui s'exécute
en moins d'une heure.

Ce n'est pas parce que quelque chose est facile à optimiser que cela signifie
que l'énergie dépensée vaut le résultat. L'ignorance est parfois une méthode
de développement parfaitement valide.

Considérez ceci comme une optimisation de *votre* temps

Vous avez la possibilité de choisir quoi et quand optimiser. A vous de
faire glisser le curseur entre "rapidité d'exécution" et "rapidité de déploiement".

Les gens entendent et répète sans fin que "les optimisation prématurés sont la source
de tous les défauts", mais ils entendent trop peu la citation complète.

> "Les programmeurs gaspillent énormément de temps à réfléchir, ou s'inquiéter,
de la rapidité de chemins d'exécution non-critique dans leurs programmes,
ces tentatives d'amélioration ont en fait un impact très négatif sur les capacités
de débuggage et de maintenance. Nous devrions oubliez les améliorations non-essentielles,
dans 97% des cas: les optimisation prématurés sont la source
de tous les défauts. Cependant nous ne devrions pas passer à côté des 3% essentiel."
>
> -- <cite>Knuth</cite>

Ajout: https://www.youtube.com/watch?time_continue=429&v=RT46MpK39rQ
   * N'ignorez pas les optimisations rapide à implémenter
   * connaître les algorithmes et les structures de données facilite l'optimisation

Devriez vous optimisez ?
> "Oui, mais seulement si le problème est impactant, le programme
est lamentablement lent, et qu'il y a des assurances que la validité,
la robustesse et clarté peuvent être maintenues."
>
> -- <cite>The Practice of Programming, Kernighan and Pike</cite>

L'optimisation prématurée peut aussi vous jeter dans des situations inextriquable.
Lorsque les exigences sont modifiées un code optimisé peut être plus difficile à modifier
et à retirer lorsque cela devient nécessaire.

[Bitfunnel une estimation de la performance](http://bitfunnel.org/strangeloop)
contient quelques chiffres représentatif de ces décisions. Imaginez un hypotétique
moteur de recherche qui a besoin de 30 000 machines réparties sur plusieurs datadenters.
Ces machines ont un coût d'exploitation approximatif de 1 000 USD par an.
Si vous êtes capable de doubler la rapidité d'exécution du logiciel, cela peut
économiser 15 millions d'USD par an. Un programmeur dépensant un an de travail
pour améliorer les performances d'un seul pourcent pourra se rentabiliser.

Dans la plupart des cas, la taille et la rapidité d'un programme n'est pas un
problème. La meilleure optimisation est de ne pas en faire. La seconde meilleure
optimisation est d'acheter un meilleur matériel.

Dès lors que vous êtes décidé à modifier votre programme, continuez cette lecture.

## Comment optimiser

### Flux de travail de l'optimisation

Avant de nous lancer dans les détails, parlons du processus général du travail
d'optimisation.

L'optimisation est une forme de refactoring. Cependant à chaque étape, plutôt
que d'améliorer tel ou tel aspect du code source (duplication de code, clareté, etc),
cela améliore tel ou tel aspect de la performance: usage du CPU, consommation mémoire,
latence, etc. Cette amélioration est généralement introduite au détriment
de la clarté. En conséquence vous devrez être équipé de bons tests unitaires (afin
de vous assurer qu'aucune regression ne soit introduite durant ce processus),
mais aussi d'un banc d'essai de mesures de performance afin de vous assurer
que les changements appliqués ont bien l'effet désiré. Vous devez être capable
de vérifier que vos changements améliorent *véritablement* la consommation CPU.
De temps en temps un changement que vous pensiez adéquat pour améliorer la performance
se révèlera contre-productif. N'hésitez pas à annulez vos changements dans ce cas là.

<cite>[What is the best comment in source code you have ever encountered? - Stack Overflow](https://stackoverflow.com/questions/184618/what-is-the-best-comment-in-source-code-you-have-ever-encountered)</cite>:
<pre>
//
// Dear maintainer:
//
// Once you are done trying to 'optimize' this routine,
// and have realized what a terrible mistake that was,
// please increment the following counter as a warning
// to the next guy:
//
// total_hours_wasted_here = 42
//
</pre>

Les bancs d'essais que vous utiliserez doivent être corrects et fournir
des résultats reproductible. Si une exécution individuelle retourne un résultat
avec trop de variance, cela rendra plus difficile de réaliser les petites
améliorations. Vous devrez utiliser [benchstat](https://golang.org/x/perf/benchstat)
ou un outil de test statistique équivalent afin de vous assurer de la validité de
vos résultats.
(Notez qu'utiliser un outil statistique est de toutes manières une bonne idée)
Les étapes pour exécuter le banc d'essai doivent être documentées, les scripts
et outils nécessaire à cette tâche doivent être sauvegardé dans vos gestionnaire
de sources avec les instructions pour s'en servir.
Prenez gardes aux grands bancs d'essais qui demandent beaucoup de temps
à s'exécuter: cela ralentira les itérations de développement.

Notez aussi que tous ce qui peut être peut être optimisé. Assurez vous de mesurer
le bon élément.

La prochaine étape est de décider ce que vous allez optimiser. SI le but est d'améliorer
l'utilisation du CPU, quel serait le gain acceptable ? Voulez vous améliorer les
perfomances par un facteur de 2 ? de 10 ? Pouvez formuler ainsi "un problème
de taille N en moins de T temps" ? Tentez vous d'améliorer la consommation de
mémoire ? De combien ? Quel perte de performance d'exécution est acceptable pour
réduire l'utilisation mémoire ? Qu'êtes vous prêt à sacrifier pour une
empreinte mémoire réduite ?

Optimisez la latence d'un service est une mission plus délicate. Des livres
entiers sont dédiés aux techniques d'optimisations et de tests de performances
de serveurs web. La première difficulté est que pour une fonction donnée, les
performances sont consistantes selon les entrées fournies. Pour un service web,
il ne s'agit pas d'une seule fonction. Un banc d'essai de service web correct
devra fournir un résultat de la distribution de la latence pour un nombres de
requêtes par seconde donné. Cet conférence fournit une bonne vue d'ensemble
des difficultés de cette tâche: ["How NOT to Measure Latency" by Gil Tene](https://youtu.be/lJ8ydIuPFeU)

TODO: See the later section on optimizing web services

Les objectifs de l'optimisation doivent être précis. Vous servez (presque)
toujours capable d'améliorer un code source. L'optimisation consiste souvent
à supprimer des retours de fonction. Vous devez savoir quand vous arrêtez. Combien
d'efforts seront encore nécessaire à optimisez le dernier bit. A quel point est-ce
acceptable de générer un code plus difficile à lire et à maintenir pour atteindre
votre objectif.

La conférence de Dan Luu précédemment mentionnée [BitFunnel performance
estimation](http://bitfunnel.org/strangeloop) fournit l'exemple de quelques
calculs approximatifs pour déterminer si le jeu en vaut la chandelle.

TODO: Programming Pearls has "Fermi Problems".  Knowing Jeff Dean's slide helps.

Lors de l'écriture d'un nouveau projet vous ne devriez pas remettre à plus tard
l'écriture de bancs d'essais. C'est facile de dire "je le ferais plus tard",
mais lorsqu'il s'agit de performance il faudra prendre cela en considération
dès le début. Vous prendriez de trop gros risque à introduire d'important
changement d'architecture pour corriger les performances en fin de projet.
Notez que *durant* le développement vous devrez vous concentrer à une conception
raisonnable, les algorithmes et les structures de données. L'optimisation bas-niveau
doit être effectué plus tard durant le cycle de développement lorsque le système
est stabilisé et qu'une vue d'ensemble des performances est disponible. Toute
tentative d'analyse et de mesure des performances d'un système incomplet ne pourra
fournir que des résultats trompeur sur la véritable source des ralentissements du
système finalisé.

TODO: How to avoid/detect "Death by 1000 cuts" from poorly written software.
Solution: "Premature pessimization is the root of all evil".  This matches with
my Rule 1: Be deliberate.  You don't need to write every line of code
to be fast, but neither should by default do wasteful things.

> "Une pessimisation prématurée advient lorsque vous écrivez du code qui est plus lent
que nécessaire, habituellement en effectuant des calculs inutile, alors que le code
équivalent plus complexe devrait couler de vos doigts naturellement."
>
> -- <cite>Herb Sutter</cite>

La mesure de performance en utilisation l'intégration continue est difficile
à cause du bruit généré par l'ensemble des tâche de tests parallélisée.
Un facteur difficile à constater dans les résultats de tests. Une bonne manière
est de faire effectuer les tests de performance par le développeur sur sa machine
(avec un matériel adéquat) et d'inclure les résultats directement dans le message
de commit réalisé à ce sujet. En ce qui concerne les changements usuels, apprenez
à détecter les mauvaises pratique à la lecture du code.

TODO: how to track performance over time?

Ecrivez du code que vous pouvez mesurer. Des profilages que vous pouvez
appliquer sur de plus large systèmes. Mesurer des pièces de codes isolées. Vous
devez être capable d'extraire et d'installer suffisamment de contexte qui permettent
l'implémentation de mesure qui sont représentative.

L'étendue entre vos performances actuelles et vos objectifs peut aussi vous
fournir une idée des actions à mener. Si vous avez seulement besoin de 10-20%
d'amélioration des performances, vous pourrez probablement vous contentez de quelques
petites modifications. Cependant si vous recherchez un facteur 10 ou plus,
alors le remplacement de quelques multiplication par des décalages binaires
ne vous permettra vraisemblablement pas d'atteindre vos objectifs. Cela
nécessitera sûrement de revoir la conception du logiciel et de modifier
en profondeur l'architecture du programme tout en gardant en tête les objectifs
de performances.

Un bon travail d'amélioration des performance nécessite un large éventail
de connaissance informatique, de la conception logiciel, au réseau, du matériel
(CPU, cache, stockage), algorithmes, configuration et débuggage. Avec des ressources
et du temps limité, prenez à bien identifier ce qui vous donnera le plus
d'améliorations: Ce ne sera pas toujours l'algorithmie ou la modification de votre
programme.

En général, les optimisations devraient être effectuées de haut en bas. Les
optimisations au niveau du système auront plus d'effet que la modification
d'expressions. Faites attention à résoudre le problème adéquatement.

Ce livre parle essentiellement de l'amélioration de la consommation
CPU, la réduction de l'utilisation mémoire et la réduction de latence.
Il faut comprendre que vous ne pourrez que trop rarement optimiser tout cela
en même temps. Peut être que votre programme s'exécute désormais plus rapidement,
mais il consomme plus de mémoire. Peut être aurez vous pu réduire la consommation mémoire,
mais désormais le programme s'exécute plus lentement.

La [loi d'Amdahl](https://en.wikipedia.org/wiki/Amdahl%27s_law) nous dit de
nous concentrer sur les goulets d'étranglement. Si vous améliorez une portion
du programme qui ne consomme que 5% du temps d'exécution, vous n'obtiendrez que
2.5% d'amélioration du temps d'exécution total. Autrement, améliorer une routine
qui consomme 80% du temps d'exécution par seulement 10% réduit la durée d'exécution
de presque 8%. Le profilage de votre programme vous permettra de déterminer les
goulots d'étranglement.

Lors du travail d'optimisation, vous voulez améliorer l'utilisation du CPU.
Un "Tri rapide" (quicksort) est plus rapide qu'un "Tri à bulles" (bubble sort)
car il résout le même problème (tri) avec moins d'opérations. C'est un algorithme
plus efficace.

Modifier les réglages d'un programme, comme l'activation des optimisations du
compilateur, ne produisent généralement que de petit gain sur l'exécution total.
Les gains significatif sont généralement obtenus par des changements algorithmique
ou l'utilisation de données de structure plus adaptées, ou des modifications
fondamentale dans la conception du logiciel. Les techniques de compilation s'améliorent,
mais trop lentement. La [loi de Proebsting](http://proebsting.cs.arizona.edu/law.html)
énonce que les améliorations des techniques de compilations doublent la rapidité
d'exécution tous les 18 *ans*, un contraste édifiant avec la (quelque peu mal comprise)
loi de Moore qui énonce la performance des processeurs double tous les 18 *mois*.
Les modifications algorithmiques ont un bien plus grand impact. Les algorithmes
sur les nombres entiers mixte [ont connus un facteur d'amélioration de 30 000 entre
1991 et 2008](https://agtb.wordpress.com/2010/12/23/progress-in-algorithms-beats-moore%E2%80%99s-law/).
En tant qu'exemple de cas réel, prenez ce [cas d'étude](https://medium.com/@buckhx/unwinding-uber-s-most-efficient-service-406413c5871d) de remplacement d'un algorithme de recherche geo-spatial par
recherche bute (brute-force) par un algorithme spécialisé pour ce type de tâche.
Il n'y a pas de remplacement de compilateur ou de réglages qui vous donneront
ce genre de résultats.

TODO: Optimizing floating point FFT and MMM algorithm differences in gttse07.pdf

Le profilage d'un programme vous montrera dans quelle portion du code
le plus de temps est dépensé. Cela peut être dans une routine coûteuse, ou
au contraire une petite routine appelée trop souvent. Plutôt que d'optimiser
cette routine, étudiez la manière de moins l'appeler, ou de la
supprimée purement et simplement. Nous discuterons des techniques d'optimisations
spécifiques dans la prochaine section.

Les 3 questions préalable à l'optimisation:

* Devrions nous seulement tenter de l'optimiser ? Le code le plus rapide est celui
qui ne s'exécute pas.
* Si oui, est ce le meilleur algorithme ?
* Si oui, est ce la meilleure *implémentation* de cet algorithme ?

## Tips d'optimisation concret

Dans son livre de 1982 "Writing Efficient Programs" John Bentley
approchait l'optimisation d'un programme comme une tâche d'ingéniérie:
Evaluer. Analyser. Améliorer. Vérifier. Recommencer. Nombre des
améliorations qu'il à mis en lumières sont désormais
implémentées automatiquement par les compilateurs. Le travail d'un
programmeur est d'appliquer les transformations qu'un compilateur
*ne peut* appliquer automatiquement.

Voici des résumés de ce livre

* <http://www.crowl.org/lawrence/programming/Bentley82.html>
* <http://www.geoffprewett.com/BookReviews/WritingEfficientPrograms.html>

Et les règles de l'optimisation d'un programme:

* <https://web.archive.org/web/20080513070949/http://www.cs.bell-labs.com/cm/cs/pearls/apprules.html>

Lorsque vous réfléchissez aux changements à appliquer à votre programme,
il y a deux grandes manières de l'aborder:
Vous pouvez modifier les structures de données ou bien changer le code.

### Changement des structures de données

Modifier les structures de données signifie ajouter ou modifier la représentation
des données que votre programme manipule. Du point de vu de la performance,
ces changements vont modifier la valeur de O() associées aux différents aspects
de vos structures de données.

Idées pour augmenter vos structure de données:

* Champs supplémentaires

  L'exemple le plus classique est de stocké la taille d'une liste liée dans un
  champ du noeud racine. Cela nécessite un peu plus de travail pour garder
  cette valeur à jour, cependant requêter la taille est beaucoup plus simple
  et efficace qu'une recherche O(n) traversant le noeud. Vos structure de données
  pourrait bénéficier d'augmentations similaires: de petites opérations de
  sauvegarde au bénéfice de meilleures performance lorsque l'opération intervient
  souvent.

  Similairement, stocké un pointeur vers des noeuds fréquemment demandés au lieu
  d'effectuer des recherches additionnels. Cela peut s'utiliser par exemple dans
  les liens "arrières" dans les listes doublement liées (doubly-linked list)
  afin que la suppression d'un noeud s'exécute en O(1). Certaines listes à
  enjambement (skip list) fournissent un "index de recherche" (search finger),
  qui vous donne accès à votre position en supposant que c'est un bon point
  de départ pour vos futurs opérations.

* Index de recherche supplémentaires

  La plupart des structures de données sont conçues pour un seul type requête.
  Si vous avez besoin d'effectuer différents types de requêtes, disposer d'une
  "vue" supplémentaire de vos données peut être une amélioration significative.
  Par exemple, un ensemble de structures pourrait avoir un entier Identifiant
  Primaire (Primary Key) que vous utilisez pour effectuer des recherches dans une
  liste, cependant de temps à autres vous nécessitez d'Identifiant Secondaire
  (secondary ID) au format texte. Plutôt que d'exécuter des recherches complète sur votre liste,
  vous pourriez sauvegarder une table de hachage pour enregistrer les correspondances
  texte vers les identifiants primaires, ou directement les structures.

* Sauvegarder des informations supplémentaires

  Par exemple, maintenir un filtre de bloom de tous les éléments insérés dans
  une liste  peut être utile pour déterminer qu'un élément n'est pas contenu
  dans la liste. Cela doit être petit et rapide afin de ne pas surcharger
  le reste de la structure de données. (Si une recherche dans votre structure
  est peu coûteuse, l'ajout d'un filtre de bloom est probablement contre-productif)

* Si les requêtes sont coûteuse, ajouter un cache

  A un plus haut niveau, un cache interne ou externe (comme memcache) peut aider.
  Cette technique peut être excessive pour une unique structure de données.
  Nous en reparlerons dans les chapitres suivant.

Ce genre de changements sont très utile si vos données sont facile à stocker
et mettre à jour.

Ceux ci sont des exemples de "travaillez moins" au niveau de la structure de données.
Tous ces exemples coûtent de l'espace mémoire. Bien souvent lorsque vous optimisez
l'utilisation du CPU, cela vous demandera d'utiliser plus d'espace mémoire.
C'est le cas classique du [compromis temps-mémoire](https://en.wikipedia.org/wiki/Space%E2%80%93time_tradeoff).

Si votre programme utilise trop de mémoire, il est aussi possible d'aller dans
l'autre sens. Réduire l'occupation mémoire en échange d'une consommation CPU accrue.
Plutôt que de sauvegarder des informations, re-calculer à chaque fois. Il est aussi
possible de compresser et de décompresser les données à la volée lorsque vous en
avez besoin.

[Small Memory Software](http://smallmemory.com/book.html) est un livre disponible
en ligne qui couvre les technique pour réduire l'utilisation mémoire de vos programmes.
Bien que celui ci fût écrit pour les programmeurs de systèmes embarqués, les
idées qu'il développent sont applicable pour les programmes exécutés sur
des systèmes modernes dotés de grosses capacités mémoire.

* Réarrangez vos données

  Elminez le bourrage (padding) de structure. Supprimez les champs superflues. Utilisez
  une plus petite structure de données.

* Utilisez une structure de données plus lente

  Les structures de données plus simple ont fréquemment une plus petite
  empreinte mémoire. Par exemple, opter pour un slice et une recherche linéaire
  plutôt qu'un arbre requérant de nombreux pointeurs.

* Format de compression personnalisé pour vos données

  L'efficacité des algorithmes de compression est dépendante de la nature de vos données.
  Il faudra choisir l'algorithme le plus adapté. Vis vous manipulez des tableaux
  de byte ([]byte), quelque chose comme snappy, gzip, lz4 fournissent de bons résultats.
  Pour les données à virgules flottantes et séries chronologique préférez go-tsz,
  pour les données scientifique référez vous à fpc. De nombreuses recherches
  existent au sujet de la compression des entiers, particulièrement dans le domaine
  des moteurs de recherche. Ceci inclut le codage différentiel (delta encoding),
  varint ainsi que des méthodes plus complexe tels le codage d'Huffman. Vous pouvez
  aussi fournir votre propre algorithme de compression spécialement optimisé pour
  vos données.

  Avez vous besoin d'inspecter vos données ou bien peuvent elles rester compressée ?
  Avez vous besoin d'un accès aléatoire ou juste d'une lecture en flux (streaming) ?
  Si vous avez besoin d'accéder aux entrées individuelles sans décompresser
  l'intégralité de vos données, vous pouvez compresser vos données par petits
  bloques et maintenir un index de localisation. Ainsi vous n'auriez qu'a consulter
  l'index pour retrouver une données particulière et ne décompresser que le bloque
  correspondant.

  Si vos données sont sur stockage persistant, que se passera t'il lors de
  la migration de celles-ci ou de l'ajout/suppression de champs ?
  If your data is not just in-process but will be written to disk, what about
  data migration or adding/removing fields. vous allez maintenant avoir à
  gérer des tableaux de byte brute au lieu de types Go structurés, vous
  devrez donc utiliser le package unsafe et considérez les formats de sérialisation.

Nous parlerons de la disposition des données plus tard.

L'informatique moderne et la hiérarchie des mémoires font que le compromis temps-
mémoire est moins évident. Il est très facile pour les tables de résultat d'être
"éloignée" en mémoire (donc plus coûteuse à accéder) et donc de justifier
un re-calcul de la valeur recherchée lorsque c'est nécessaire.

Cela signifie aussi que les bancs d'essais montreront des améliorations
qui ne sont pas constatés dans les systèmes en production, cela est dû à un
conflit de cache (les tables de résultats sont présente dans le processeur durant
l'exécution des bancs d'essai, mais se trouvent être évacués par d'autres données
en production).

Le papier Google [Jump Hash](https://arxiv.org/pdf/1406.2294.pdf), démontre
cela en comparant les performances d'un cache de processeur libre et disputé.
(Voir les graphiques 4 et 5 du papier

TODO: how to simulate a contended cache, show incremental cost

Un autre aspect à prendre en compte est le temps de transfert de vos données.
Habituellement les accès disque et réseaux sont très lent, charger des bloques
de données compressés est donc plus rapide même avec la durée de décompression
supplémentaire consommée par le CPU. Comme toujours, mesurez. Un format binaire
est généralement plus rapide et léger qu'un format texte, mais cela le rend
illisible par un humain.

Pour le transfert de données, choisissez un protocole moins verbeux, ou
améliorez vos API pour permettre l'exécution de requêtes partiel. Par exemple,
utilisez des requêtes incrémental plutôt que de forcer le chargement complet
du jeu de données.

### Changement algorithmique

Si vous ne modifiez pas les données, alors vous devez changer votre code.

La meilleure amélioration viendra probablement d'un changement algorithmique.
C'est la même chose que de remplacer un tri à bulle (`O(n^2)`) par un tri rapide
(`O(n log n)`) ou de remplacer une recherche linéaire dans un tableau (`O(n)`)
par un arbre binaire (`O(log n)`) ou dans une table de hachage (`O(1)`).

Voici comment un logiciel devient lent. Des structures originellement
conçues pour un certain usage sont ré utilisées pour quelque chose d'autre.
Cela se produit graduellement.

C'est important d'avoir une compréhension intuitive des différents niveaux
de complexités (big-O). Choisissez la bonne structure de données pour résoudre
vos problèmes. Vous n'avez pas toujours besoin de sauver des cycles cpu,
mais cela permet de prévenir des problèmes de performances lamentablement évidents
qui ne pourrait être détectés que bien plus tard.

Les classes de complexités sont:

* O(1): L'accès à un champ, un tableau ou une table de hachage

  Conseil: ne vous en souciez pas trop (mais gardez en mémoire le facteur constant)

* O(log n): recherche binaire

  Conseil: c'est un problème si c'est utilisé dans une boucle

* O(n): simple boucle

  Conseil: vous faites cela tout le temps

* O(n log n): diviser et conquérir, tri

  Conseil: Cela reste plutôt rapide

* O(n\*m): boucle imbriquée / quadratique

  Conseil: prenez garde et restreignez la taille de vos données

* N'importe quoi d'autres entre quadratique et subexponentiel

  Conseil: n'exécutez pas cela sur un million de ligne

* O(b ^ n), O(n!): exponentiel et plus

  Conseil: bonne chance si vous avez plus d'une douzaine ou deux douzaine de données

Lien: <http://bigocheatsheet.com>

Imaginons que vous ayez à rechercher dans un jeu de données non triés. Vous
pourriez penser de prime abord "Je devrais utiliser un arbre binaire", sachant
qu'ils implémentent une recherche en O(log n) qui est plus rapide qu'une
recherche linéaire O(n). Cependant, une recherche binaire nécessite que les données
soient triées, ce qu'il faudra donc réaliser au préalable et qui vous coûtera
O(n log n). Si vous effectuez beaucoup de recherches, ceci pourrait être compensé.
Cependant, si vous réalisez essentiellement des recherches, peut être que l'utilisation
d'un arbre binaire n'est pas le meilleur choix et que vous feriez
mieux d'utiliser une table de hachage O(1).

Etre capable d'analyser votre problème en termes de complexité vous permettra
de déterminer si vous avez atteint les limites de ce qu'il est possible de faire
avant de rechercher d'autres méthodes pour accélérer les choses. Par exemple,
trouvez le minimum d'une liste non tries est `O(n)`, car vous devez vérifier tous
les éléments. Vous ne ferez pas mieux.

Si votre structure de données est statique, alors il est possible de faire beaucoup mieux
que lorsque celles ci sont dynamique. C'est plus facile de construire une solution
optimisée pour répondre exactement à vos recherches. Une fonction de hachage minimale
parfaite (minimal perfect hashing) peut être adéquat, ou bien un filtre de bloom
pré calculé. Cela sera aussi judicieux si votre structure de données est modifiée
assez peu souvent pour amortir le coût de construction.

Vous devriez choisir la structure de données la plus simple et raisonnable.
C'est le béaba du génie logiciel pour écrire des programmes rapide. Ce devrait être
votre manière de penser par défaut. Si vous n'avez pas besoin d'un accès aléatoire,
n'utilisez pas une liste-liée. Si vous savez que vous devez parcourir vos
données dans l'ordre, n'utilisez pas une table de hachage.
Les exigences changent et vous ne pourrez pas toujours les deviner à l'avance.
Faites le meilleur choix en fonction de la charge.

<http://daslab.seas.harvard.edu/rum-conjecture/>

Les structure de données utilisées pour un problème similaire diffèrent
pour une opération identique. Un arbre binaire tri les données lors de
l'insertion. Un tableau non trié est rapide à écrire mais c'est non trié: à la
fin vous devez faire le travail de tri.

Lorsque vous écrivez un paquet pour des tiers, ne succombez pas à la tentation
d'optimiser au préalable pour tous les cas d'usage. Cela produira du code non lisible.
Les structure de données spécifique ne sont efficace que dans un cas spécifique.
Vous ne pourrez jamais faire de la télépathie ou prédire le futur. Si un utilisateur
vous dit "Ton paquet est trop lent pour ce cas d'usage", vous pourriez lui répondre
"Utilises cet autre paquet". Un paquet devrait "faire une chose et le faire bien".

De temps en temps une structure hybride peut vous donnez les résultats dont vous
avez besoin. Par exemple, en groupant vos données vous pourriez améliorez vos
requêtes. En théorie vous paierez toujours une complexité O(n), mais la constante
sera plus petite. Nous revisiterons ce genre d'améliorations lorsque nous
parlerons des réglages logiciels.

Deux chose qui sont souvent oubliée lorsque l'on discute de complexité:

En premier, il y à toujours un facteur constant. Deux algorithmes qui ont
la même complexité algorithmique peuvent avoir facteur constant différent.
Imaginez boucler sur une liste une centaine de fois plutôt que de boucler qu'une
seule fois. Même si chacune de ces opérations ont une complexité O(n), l'un
possède un facteur constant 100 fois supérieur.

Ces facteurs constant expliquent pourquoi bien que le tri fusion (merge sort),
ou le tri par tas (heapsort) sont en O(n log n), tout le monde utilise le tri
rapide (quicksort) car c'est le plus efficace. Il a le plus petit facteur constant.

La seconde chose est que la notation de complexité (big-O) ne dit que cela "tant
que n grandit jusqu'à l'infini". Cette notation parle de l'augmentation de la taille
du jeu de donnée, "Tant que la taille augmente, voici le coût que vous paierez à
l'exécution". Cette notation ne dit rien des performances réels que vous obtiendrez,
ni des impacts sur une petite valeur de n.

Il y à fréquemment un point de rupture en dessous duquel où un algorithme naïf
est plus rapide. Un bon exemple tiré de la librarire dtandard du langage Go
est le paquet `sort`. La plupart du temps celui ci utilise le tri rapide,
mais il réalise une passe de tri par shell puis un tri par insertion lorsqu'il
détecte qu'il y a moins de 12 partitions.

Pour certains algorithmes, le facteur constat peut être tellement grand
que ce point de rupture peut être plus grand que toutes entrées raisonnable.
C’est-à-dire que l’algorithme O(n ^ 2) est plus rapide que l'algorithme O(n)
pour toutes les entrées que vous êtes susceptible de jamais traiter avec.

Cela signifie que vous devez être capable de déterminer la taille de vos
données d'entrées, à la fois pour choisir l'algorithme le plus approprié
mais aussi pour écrire de bon bancs d'essais.
10 éléments ? 1000 éléments ? 1000000 d'éléments ?

C'est aussi valable en sens inverse: Vous pourriez choisir un algorithme plus
compliqué pour obtenir une complexité mise à l'échelle O(n) au lieu de O(n^2),
même si vos bancs d'essais sur une entrées de taille plus petite fournit des
résultats plus lent. Cela s'applique aussi à la plupart des structures de données
sans verrouillage (lock-free). Celles ci sont généralement plus lente lorsqu'un
seul fil d'exécution les consomme mais elles se comportent bien mieux lorsque
plusieurs fils d'exécution les utilisent.

La hiérarchie des mémoires de l'informatique moderne complique un peu les choses,
en cela que les caches préfèrent les accès prédictible du balayage d'un tableau
plutôt que l'accès aléatoire de la résolution d'un pointeur. Il reste préférable
de commencer avec un bon algorithme. Nous parlerons plus tard de cela dans
la section spécifique au matériel.

> La lutte ne peut pas toujours aller au plus fort, ni la course au plus rapide,
mais c'est la manière de parier.
> -- <cite>Rudyard Kipling</cite>

Parfois le meilleur algorithme pour un problème spécifique n'est pas d'utiliser
un seul algorithme, mais plusieurs algorithmes spécialisés pour des entrées
légèrement différentes. Ces "polyalgorithmes" détectent rapidement le type d'entrées
qu'ils reçoivent et choisissent alors le meilleur algorithme de traitement.
C'est ce que le paquet sort susmentionné implémente: déterminer la taille du
problème puis choisir le meilleur algorithme. En plus de combiner le tri rapide,
le tri de shell et le tri par insertion, il traque la profondeur de récursion
pour utiliser un tri par tas lorsque nécessaire. Les paquets `strings` et `bytes`
font quelque chose de similaire, ils identifient la casse et spécialisent les algorithmes.
Comme pour la compression de données, plus vous en saurez sur vos données d'entrées,
plus vous serez capable d'utiliser le meilleur algorithme. Même si une optimisation
n'est pas toujours applicable, la complexité induite d'identification et l'exécution
d'une logique alternative peut être justifiée.

Cela s'applique aussi aux sous problèmes que vos algorithmes doivent résoudre.
Par exemple être capable d'utiliser un tri par base (radix sort) peut être
significativement bénéfique pour les performances, ou bien encore d'utiliser
l' algorithme de sélection de Hoare (quickselect) si vous n'avez besoin que
d'un tri partiel.

Parfois plutôt qu'une spécialisation pour votre tâche particulière, le
plus approprié est de l'abstraire dans un espace de problème plus générale
qui est bien étudié par les chercheurs. Vous pourrez ainsi appliquer la solution
plus générale à votre problème. Cette transformation peut être un gain significatif.

Similairement, utiliser un algorithme plus simple signifie que les compromis,
l'analyse et les détails d'implémentation  ont plus de chances d'être plus étudiés
et mieux compris que des implémentation exotique ou ésotérique.

Des algorithmes plus simples peuvent aussi être plus rapide. Ces deux exemples
ne sont pas isolés
  https://go-review.googlesource.com/c/crypto/+/169037
  https://go-review.googlesource.com/c/go/+/170322/

TODO: notes on algorithm selection

TODO:
  improve worst-case behaviour at slight cost to average runtime
  linear-time regexp matching
  randomized algorithms: MC vs. LV
    improve worse-case running time
    skip-list, treap, randomized marking,
    primality testing, randomized pivot for quicksort
    power of two random choices
    statistical approximations (frequently depend on sample size and not population size)

 TODO: batching to reduce overhead: https://lemire.me/blog/2018/04/17/iterating-in-batches-over-data-structures-can-be-much-faster/

## Entrées de banc d'essai

Des entrées représentative correspondent rarement au pire cas théorique.
Le banc d'essai est vital pour comprendre le comportement de votre système en
production.

Vous devez connaître la nature des données d'entrée que votre système recevra
après son déploiement, et vos mesures doivent être exécutés à l'encontre de jeux
de données tirées de la même source. Comme nous l'avons vu précédemment, les
algorithmes ne sont pas efficace pour différentes taille d'entrées. Si vous
vous attendez à recevoir une taille inférieure à 100, alors vous devez construire
un banc d'essai reflétant cela. Dans le cas contraire, le résultat ne sera pas
optimum si vous avez opté pour un algorithme efficace pour une entrée très large n=10^6.

Soyez capable de générer un jeu de données représentatif. Une distribution
différente de vos données peuvent provoquer des résultats différents: pensez
au classique "la complexité d'un tri rapide est O(n^2) lorsque les données sont
triées". De la même manière, une recherche par interpolation est O(log log n)
pour des données uniformément distribuées, mais O(n) dans le pire des cas contraire.
Connaître la nature de vos entrées est la clef pour la sélection du meilleur
algorithme et à l'écriture d'un bon banc d'essai. Si votre jeu de données est
non représentatif vous finirez par optimiser votre programme pour un jeu de données
trop spécifique.

Ceci implique que vos données de banc d'essai doivent être représentative d'un
vrai cas d'usage en production. N'utilisez que des données aléatoire pourrez biaiser
le comportement de vos algorithmes. Le cache et la compression sont deux exemples
qui exploitent un biais de distribution qui ne sont pas présent dans un jeu aléatoire,
alors qu'un arbre binaire se comporte mieux avec des données aléatoire puisqu'ils
ont tendance à garder l'arbre équilibré (par ailleurs c'est l'idée derrière l'abretas,
ou arbre binaire de recherches aléatoire, ndt: binary treap)

D'une autre manière, considérez le test d'un système de cache avec une seule
requête, elles toucheront toutes le cache en conséquence les résultats de
performances seront irréaliste en comparaison d'une charge de production qui
contiendra une grande variétés de requêtes.

Soyez conscient que certains problèmes ne sont pas détectable sur votre ordinateur
portable et ne seront visible que lors du déploiement quand votre programme
atteindra les 250 000 requêtes par seconde sur un matériel équipé de 40 processeurs.
Le ramasse-miette pourra aussi se comporter différemment lors de vos bancs d'essai
que lors de l'exécution en production.  Dans de (très) rares cas un micro banc
d'essai se montrera plus lent qu'après le déploiement. Ces micro banc d'essai
peuvent vous aider à identifier un problème, mais il vaut mieux être capable
de mesurer les performances du système déployé.

Ecrire de bon bancs d'essai est difficile

* <https://timharris.uk/misc/five-ways.pdf>

Utilisez les moyennes géométrique pour comparer des groupes d'exécution

* <https://www.cse.unsw.edu.au/~cs9242/current/papers/Fleming_Wallace_86.pdf>

Evaluez la précision d'un banc d'essai

* <http://www.brendangregg.com/blog/2018-06-30/benchmarking-checklist.html>

### Optimisation d'un programme

L'optimisation d'un programme était un art, puis les compilateurs s'améliorèrent.
Désormais les compilateurs optimisent le code simple mieux que ceux compliqués.
Le compilateur Go à encore un long chemin à faire avant d'atteindre les capacités
de gcc et clang, à cause de cela vous devez être précautionneux lorsque vous
effectuez des réglages et plus particulièrement lorsque vous procéder à la mise
à jour de votre version de Go afin que votre code reste le meilleur.
Il existe clairement des cas où les réglages appliqués pour contourner le
manque d'une certaine optimisation ont rendu le code plus lent dès lors que le
compilateur fût mis à jour.

Mon implémentation de chiffrement RC6 obtenait une amélioration de 10% pour une
boucle interne en utilisant les paquets `encoding/binary` et `math/bits` au lieu
de ma version écrite pour l'occasion.

De la même manière, le paquet `compress/bzip2` était plus performant après l'avoir
modifié pour [utiliser un code plus simple que le compilateur optimisait mieux](https://github.com/golang/go/commit/9eb219480e8de08d380ee052b7bff293856955f8)

Si vous contourner une limitation spécifique du runtime ou du code généré par le
compilateur, forcez vous à toujours documenter vos changements avec un lien
vers le ticket upstream. En faisant ainsi vous pourrez rapidement revisitez vos
changements lorsque la limitation aura disparue.

Résistez à la tentation du culte cargo "performance tips", ou même de généraliser
vos hypothèses depuis vos expériences. Chaque problème de performance mérite
d'être approché de manière unique. Même si quelque solutions a fonctionné dans le
passé, implémentez et exécutez des bancs d'essai afin d'avoir la certitude que
vos changements sont efficace. Vos expériences peuvent vous guider, mais elles ne
peuvent valider vos changements.

Le réglage d'un programme est un travail itératif. Continuez de revisiter votre
code et de détecter les changements que vous pourriez appliquer. Assurez vos arrières
en validant vos avancées. Fréquemment une amélioration vous ouvrira la porte vers
d'autres possibilités. (Maintenant que je ne fais plus A, je peux simplifier B en
faisant C). Cela requiert de toujours garder un point de vu global et de pas
rester scotcher sur quelques lignes de code en particulier.

Dès lors que vous aurez choisi un algorithme, le réglage consiste à améliorer
son implémentation. En terme de complexité (big-o notation), cela signifie
réduire les constantes associées avec votre programme.

Chaque réglage du programme consistera soit à rendre plus rapide un élément,
ou à l'exécuter moins souvent. Les changements d'algorithmes tombent aussi
dans ces catégories, mais nous allons étudier les changements plus précis.
La manière précise de faire cela varie en même temps que la technologie évolue.

Améliorer la performance d'un code peut être de remplacer SHA1 ou `hash/fnv1`
avec une fonction de hachage alternative plus rapide. Exécuter moins souvent
peut consister à sauvegarder le résultat du hachage d'un gros fichier afin
de ne pas répéter l'opération.

Commentez. Si quelque chose n'à pas besoin d'être exécuté, expliquez le.
Fréquemment lors de l'optimisation d'un algorithme vous découvrirez des étapes
qui n'ont besoin d'être exécutées que sous certaines conditions précises.
Documentez les. Quelqu'un d'autre pourrait prendre ces changements pour un bug
et les retirer.

> Un programme vide retourne la mauvaise réponse immédiatement.
>
> C'est facile d'être rapide si vous n'avez pas besoin d'être correct.

La notion de "validité" peut avoir différente signification en fonction de votre
problème. Les algorithme heuristique qui donne la réponse la plus probablement
correct la plupart du temps peuvent être rapide, tout autant que les algorithmes
qui devinent et améliorent et vous indique quand vous arrêter car vous avez atteint
une limite acceptable.

Cas d'usages du cache:

Nous sommes tous familier avec memcache, il y a aussi les caches interne à vos
processus. En utilisant un cache interne vous réduisez les coûts réseaux
et de sérialisation cependant vous augmentez la pression sur le ramasse-miette
puisqu'il y a plus de mémoire à gérer. Vous devrez aussi étudier les stratégies
d'éviction, d'invalidation et la sécurisation des appels parallèles (thread-safety).
Un cache externe gèrera l'éviction, mais vous devrez tout de même réfléchir à
l'invalidation des données. Les appels parallèles restent un problème car celui
deviendra un état mutable partagé entre différentes goroutines dans le même service,
ou bien même à travers plusieurs services si le cache est partagé.

Un cache sauvegarde les informations pour lesquels votre programme à dépensé du
temps pour leurs calculs dans l'espoir que celui ci pourra les ré utiliser et
ainsi sauvegarder quelques cycles cpu. Un cache n'a pas besoin d'être complexe.
La sauvegarde d'un élément -- le résultat de la requête la plus récente -- peut
apporter un énorme gain, comme nous le verrons dans l'exemple de `time.Parse()`.

Lors de l'utilisation d'un cache il est primordial de comparer le coût (exprimé
en durée [ndt wall-clock] et la complexité introduite) de cet ajout avec le
coût de re calcul. Un algorithme de cache qui donne de meilleurs taux d'utilisation
induit sa propre complexité. Une éviction aléatoire est une méthode simple et
rapide qui peut être efficace dans bien des cas. L'*insertion*, la mise en cache,
aléatoire peut vous aider à enregistrer les données les plus populaire. Bien que
ces techniques peuvent ne pas être aussi efficace que d'autres implémentations
plus complexes, la grosse améliorations viendra de l'ajout d'un cache tout simplement:
choisir avec minutie l'algorithme de cache n'apportera que des améliorations mineures.

C'est important de surveiller vos décision de stratégies d'éviction après vos
déploiements. Si lors de l'exécution en production vous constatez que le taux
d'utilisation est faible, il se pourrait bien que cela soit plus coûteux de garder
le cache que de re calculer les opérations. J'ai par le passé pu constater que
lorsque je testais mon cache avec des données de production celui ci n'en valait
pas la peine. Nous n'avions simplement pas suffisament de répétition de requête
pour justifier la complexité du cache.

Vos estimations attendues du taux d'utilisation sont importantes. Vous voudrez
exporter cette information vers votre système de monitoring. Un changement de ces
taux signifie que votre trafic a changé et qu'il est temps de revisiter les
paramètres de dimensionnement ou d'expiration.

Un très gros cache peut augmenter la pression sur le ramasse-miette. A l'extrême
(faible ou aucune éviction, mise en cache de toutes les exécutions d'une fonction
très coûteuse) cela devient de la [memoization](https://en.wikipedia.org/wiki/Memoization)

Optimisation d'un programme:

L'art de l'optimisation d'un programme est le travail d'amélioration itératif
par petit pas.
Elgon Elbre le décrit ainsi:

* Déterminez une hypothèse quand à la lenteur
* Supputez plusieurs solutions
* Tentez les toutes, gardez la meilleure
* Gardez la deuxième meilleure, au cas où
* Répétez.

L'optimisation peut se faire de diverses manières.

* Si possible, garder l'implémentation précédente pour comparer vos changements.
* Autrement, générer suffisamment de cas significatif pour comparer vos changements.
* "Suffisamment" signifie d'identifier les cas particuliers, comme ceux qui sont
  les plus à même d'être affecté par les objectifs que vous aurez fixé.
* Exploiter une propriété mathématique:
  * Noter que l'implémentation et l'optimisation des calculs numériques est
    un art à part entière.
    * <https://github.com/golang/go/commit/ed6c6c9c11496ed8e458f6e0731103126ce60223>
    * <https://gist.github.com/dgryski/67e6a7ff94c3a1add30eb26ec0ad8b0f>
    * multiplier avec des additions
    * utilisez WolframAlpha, Maxima, sympy et autres outils similaires pour optimiser
      ou créer des tables de recherche.
    * (Voir aussi, https://users.ece.cmu.edu/~franzf/papers/gttse07.pdf)
    * remplacer les nombres réels par des entiers
    * suppression de la racine carré de mandelbrot, ou la suppression lltb
      de valeur absolue,  `a < b/c` => `a * c < b`
    * considérer différentes représentations numérique : virgule fixe, virgule flottante,
      (de plus petit) entier
    * plus sophistiqué: des entiers avec des accumulateurs d'erreur (Algorithme de tracé de segment de Bresenham), nombres à bases multiples / système numérique redondant (ndt: mixed radix ?)
  * "payez seulement pour ce que vous utilisez, pas ce que vous pourriez utiliser"
    * aucune partie d'un tableau plutôt que l'intégralité de celui ci
  * Il vaut mieux procéder par petits ajustements, quelques déclarations à la fois
  * Vérification simple et rapide avant les vérifications plus lourde
    * e.g., strcmp avant une regexp (q.v., une filtre de bloom avant une requête)
    * "exécuter ce qui est coûteux moins souvent"
  * les cas communs avant les cas rares
    i.e., évitez les tests superflues qui échoue en permanence
  * Le déroulage reste efficace : https://play.golang.org/p/6tnySwNxG6O
    * compromis entre la taille du code et les tests de branche
  * utiliser des index d'offset plutôt que l'affectation de tranche (ndt slice assignments)
    car cela optimisera les vérifications de limites (ndt bound checking),
    dépendances des données et la génération de code (il y aura moins à copier dans
      la boucle intérieur)
  * supprimez les vérification de limite (ndt bound checks) et les tests de nil
    dans les boucles : https://go-review.googlesource.com/c/go/+/151158
  * other tricks for the prove pass (ndt ?)
  * c'est maintenant que les idées de Hacker Delight tombent

Le folklore des astuces d'optimisations pour la performance s'appuie sur des
compilateurs peu optimisés et encouragent les programmeurs à implémenter ces
optimisations à la main. Les compilateurs ont utilisé le décalage arithmétique
au lieu des multiplications ou divisions par une puissance de 2 depuis
plus de 15 ans désormais -- personne ne devrait le faire à la main. Similairement,
sortir les invariants des boucles, le débouclage de boucle simple, les suppressions
de sous-expression simples et pleins d'autres optimisations sont réalisées
automatiquement par gcc, clang et consors. Le compilateur Go implémentent
beaucoup de ces optimisations et continue chaque jour de s'améliorer. Comme toujours,
comparez avant de commit une nouvelle version.

Les transformations que le compilateur ne peut réaliser sont celles des
particularités algorithmique que vous connaissez, à propos de vos données,
des invariants de votre système, d'autres suppositions que vous pouvez faire,
et les changements qui requiert de savoir comment supprimer ou modifier des
étapes dans les structures de données.

Chaque optimisation codifie une supposition sur les données. Celles-ci *doivent*
être documentées, mieux encore, testées. Ces suppositions feront bugger,
ralentir ou crasher vos programmes au cours de leurs évolutions.

Les améliorations d'un programme sont cumulative. 5x 3% d'améliorations
résultent en 15% d'améliorations. Lorsque vous optimisez, cela vaut le coup
de penser à vos objectifs d'améliorations. Remplacer une fonction de hachage
avec une implémentation plus rapide est gain constant.

Comprendre vos exigences et où vous pouvez les altérer peut vous mener à
des améliorations de performance. Un problème soulevé sur le channel slack  
\#performance des Gophers consistez à déterminer le temps dépensé à créer
un identifiant unique d'une table de hachage composée de clefs/valeurs
de type string. La solution orignal proposée d'extraire les clefs, de les trier,
et fournir le résultat à une fonction de hachage. La solution optimisée que nous
avons pu trouver consistait à hasher les paires de clef/valeur lorsqu'elles étaient
ajoutées à la table de hachage, puis finalement d'appliquer un xor à tous les
hash pour créer un identifiant unique.

Voici un exemple de spécialisation

Supposons que nous ayez à analyser un fichier de log massif journalier,
chaque ligne commence par un horodatage

```
Sun  4 Mar 2018 14:35:09 PST <...........................>
```

Pour chaque ligne, nous utiliserons `time.Parse()` pour le transformer en époque.
Si le profilage nous démontre que `time.Parse()` est le goulot, nous avons
quelques possibilités à notre portée pour améliorer cela.

Le plus simple est de garder une ligne de cache du dernier horodatage analysé.
Tant que le fichier contient plusieurs lignes pour la même seconde, c'est
optimisé. Pour le cas d'un fichier de 10M de lignes cette stratégie réduit le
nombre d'appel au couteux `time.Parse()` à 86 400 -- un pour chaque seconde.

TODO: code example for single-item cache

Pouvons nous faire mieux ? Comme nous connaissons exactement le format d'horodatage
*et* qu'ils sont tous de la même journée, nous pouvons écrire une fonction d'analyse
spécifique. Nous pouvons calculer l'époque pour minuit, puis extraire heure, minute et
secondes de la chaîne de caractère -- tous les horodatage sont positionnés au
même index dans la chaîne de caractère -- et réaliser quelques calculs d'entiers.

TODO: code example for string offset version

Selon mes bancs d'essais, cela réduit la durée d'analyse de 275ns/op à 5ns/op.
(Bien évidemment, à 275ns/op vous serez toujours ralentit par vos I/O et
non par le coût CPU d'analyse de l'horodatage).

L'algorithme général est lent parce qu'il doit analyser plus de cas. Votre
algorithme peut être amélioré parce que vous comprenez votre problème. Cependant,
le code est désormais plus intolérant aux changements, il est intimement lié à
la structure de vos données d'entrées. Le code est aussi plus difficile à mettre
à jour si le format change.

L'optimisation est une spécialisation, un code spécialisé est plus fragile aux
changements qu'un code plus général.

L'implémentation de la librairie standard soit être suffisamment rapide pour la
plupart des cas. Si vous avez besoin de meilleurs performance vous devrez
probablement spécialisés vos implémentations.

Profiler régulièrement pour suivre et observer les caractéristiques de vos
systèmes et soyez prêt à ré-optimiser lorsque votre trafic change. Apprenez les
limites de votre système  et monitorer les métriques qui vous permettront de
prédire lorsque vous dépasserez ses limites.

Lorsque l'utilisation de votre application change, différentes parties du code
peuvent devenir chaude. Revisitez vos optimisations précédentes et évaluer les
pour déterminer si elles en valent toujours le coup, autrement retirez les pour
revenir à un code plus lisible lorsque cela est possible. J'avais un système dont
j'avais optimisé les étapes de démarrage avec un enchevêtrement complexe de mmap,
reflect et unsafe. Après que nous ayons modifié nos méthodes de déploiement,
cette partie du code n'était plus nécessaire et je l'ai remplacé par des opérations
de fichiers bien plus lisible.

### Récapitulatif du travail d'optimisation

Toutes les optimisations devraient suivre ces étapes :

1. déterminez vos objectifs de performances et confirmer que vous ne les atteignez pas
    * Cela peut être le CPU, l'allocation sur le tas (heap allocation), ou des goroutine bloquantes
2. évaluez l'amélioration de vos solutions en utilisant le framework standard
   (<http://golang.org/pkg/testing/>)
    * Soyez certains d'évaluer le bon élément sur votre OS et architecture
3. profiler encore par la suite pour vérifier que le problème à disparu
4. utilisez <https://godoc.org/golang.org/x/perf/benchstat> ou
    <https://github.com/codahale/tinystat> pour vérifier que la variance des
    résultats est suffisante pour justifier la complexité induite par l'implémentation
    de l'optimisation.
5. utilisez <https://github.com/tsenart/vegeta> pour les tests de charge de service
   http (ainsi que: k6, fortio, fbender)
   - si possible, testez la montée/descente de charge en plus de la charge constante
6. soyez certains que vos résultats de latences sont cohérent

Le premier point est important. Il vous indique où et quand commencer à optimiser.
Plus important encore, il vous indique quand vous arrêtez. La plupart des
améliorations requièrent un code plus complexe. Finalement, vous pouvez *toujours*
améliorer une implémentation. C'est toujours un compromis.

## ramasse miettes

Vous payez pour vos allocations plus d'une fois. La première fois est évidemment
lors de l'allocation. Mais aussi à chaque fois que le ramasse miettes s'exécute.

> Réduire/Réutiliser/Recycler.
> -- <cite>@bboreham</cite>

* allocations de piles et de tas
* quelles sont les causes d'une allocation sur le tas ?
* comprendre l'analyse d'échappement (et ses limitations actuelles)
* /debug/pprof/heap , et -base (ndt: -base ?)
* La conception d'API qui préviennent les allocations:
  * retourner le buffer interne afin que l'appelant puisse le réutiliser plutôt
    que de le forcer à allouer
  * vous pouvez aussi précautionneusement modifier un tableau en place pendant
    que vous le scannez
  * prendre une struct en argument peut permettre à l'appelant de l'allouer
    sur la pile
* réduire l'utilisation des pointeurs pour réduire la durée de scan du ramasse miettes
  * tableau sans pointeurs
  * tableau de hachage avec des clefs/valeurs sans pointeurs
* GOGC
* ré utilisation des buffers (sync.Pool vs spécifique go-slab, etc)
* tranche de tableau vs indexation : l'écriture d'un pointeur pendant que le ramasse
  miettes s'exécute requiert une barrière d'écriture : https://github.com/golang/go/commit/b85433975aedc2be2971093b6bbb0a7dc264c8fd
  * pas de barrière d'écriture si vous écrivez sur la pile https://github.com/golang/go/commit/2140975ebde164ea1eaa70fc72775c03567f2bc9
* utilisez des variables d'erreurs plutôt que `errors.New()` / `fmt.Errorf()` sur le site d'appel (style ou performane ? Cela requiert un pointeur, donc de toute façon la valeur va sur le tas)
* utilisez des erreurs structurées pour réduire les allocations (valeur de type struct), créez
  les chaînes de caractère lors de l'impression écran
* size classes (ndt ?)
* évitez les larges allocations en utilisant des tableaux ou chaînes de caractère plus petite

## Runtime et compilateur

* coûts d'appel par une interface (appels indirects au niveau du CPU)
* runtime.convT2E / runtime.convT2I
* assertions de type Vs commutateurs de type (ndt : type switch)
* defer
* cas d'implémentations spéciaux pour les entiers, chaînes de caractère
  * tableau de hachage pour les types byte/uint16 ne sont pas optimisés; utilisez des tableaux.
  * vous pouvez imiter un nombre réels avec `math.Float{32,64}{from,}bits`, mais faites attention aux problèmes d'égalité des nombres réels
* l'élimination des tests de limite (bound checks)
* les copies []byte <-> string, l'optimisation des maps
* les itérations de tableaux de taille fixe à deux variables (ndt: `for i,v range [0]int{}{}`)
  sont implémentées avec une copie, utilisez un tableau à taille variable (ndt: slice) :
    * <https://play.golang.org/p/4b181zkB1O>
    * <https://github.com/mdempsky/rangerdanger>
* utilisez la concaténation de chaînes de caractère plutôt que fmt.Sprintf lorsque cela
  est possible; le runtime implémente ces instructions de manières spécifique

## Unsafe

* Et tous les dangers qui l'accompagnent
* Les cas d'usages communs
* mmaper des données
  * bourrage de struct
  * pas toujours suffisamment efficace pour justifier le coût complexité/sécurité
  * mais "en dehors du tas", dnc ignoré par le ramasse miettes (mais un tableau sans pointeur aussi)
* vous devrez réfléchir au format de sérialisation: comment gérer les pointeurs, l'indexation (mph, index header)
* desérialisation rapide
* protocole de lien binaire vers une struct lorsque vous avec déjà un buffer
* les conversions chaînes de caractère <-> tableaux, []byte <-> []uint32, ...
* l'astuce non sécurisée d'utilisation de booléens plutôt que des entiers (cependant
  référez vous à cmov) (notez aussi que != 0 ne génère pas d'opération de branchement)
* bourrage
  - https://dave.cheney.net/2015/10/09/padding-is-hard
  - http://www.catb.org/esr/structure-packing/#_go_and_rust
  - https://golang.org/ref/spec#Size_and_alignment_guarantees
  - https://github.com/dominikh/go-tools structlayout, structlayout-optimize
  - write tests for struct layout with unsafe.Offsetof to notice breakage from unsafe or asm
  - écrivez des tests pour vos dispositions de struct avec unsafe.Offsetof pour
    identifier une rupture de unsafe ou d'assembleur

## Pièges usuels avec la librairie standard

* `time.After()` fuit tant qu'il n'a pas émit; utilisez `t := NewTime(); t.Stop() / t.Reset()`
* Réutilisations des connections HTTP...; s'assurer que le corps de la requête est drainée (issue #? ndt: \#249 \#26095)
* `rand.Int()` et compagnie sont 1) protégés par un `mutex` et 2) coûteux à créer
  * envisager un générateur de nombre aléatoire alternatif (go-pcgr, xorshift)
* `binary.Read` and `binary.Write` utilisez sont lent car ils utilisent de la réflexion (ndt: reflect); faites le manuellement (https://github.com/conformal/yubikey/commit/613e3b04ae2eeb78e6a19636b8ff8e9106d2e7bc)
* utilisez le paquet `strconv` plutôt que `fmt`
* ...

## Implémentations alternative

Liste de remplacements populaires de la librairie standard

* `encoding/json` -> [ffjson](https://github.com/pquerna/ffjson), [easyjson](https://github.com/mailru/easyjson), [jingo](https://github.com/bet365/jingo) (uniquement l'encodeur), etc
* `net/http`
  * [fasthttp](https://github.com/valyala/fasthttp/) (API incompatible, imparfaitement compatible avec la RFC)
  * [httprouter](https://github.com/julienschmidt/httprouter) (contient des fonctionnalités supplémentaire autre que la performance; Je n'ai jamais vu le routage dans mes profiles)
* regexp -> [ragel](https://www.colm.net/open-source/ragel/) (ou autre paquet d'expression régulière)
* serialisation
  * `encoding/gob` -> <https://github.com/alecthomas/go_serialization_benchmarks>
  * protobuf -> <https://github.com/gogo/protobuf>
  * tous les formats de sérialisation font des compromis: choisissez celui qui correspond à vos besoins
    - grosse charge de travail en écriture -> choisissez un format d'encodage efficace
    - grosse charge de travail en lecture -> choisissez un format de décodage efficace
    - D'autres considérations -> taille de l'encodage, compatibilité avec vos langages/outils
    - compromis d'un format binaire ou d'un format texte auto-descriptif
* `database/sql` -> contient des compromis qui affectent les performances
  *  recherchez un pilote qui ne l'utilise pas: `jackx/pgx`, `crawshaw sqlite`, ...
* gccgo (benchmark!), gollvm (WIP)
* conteneur/liste: utiliser plutôt un slice (presque tout le temps)

## cgo

> cgo is not go
> -- <cite>Rob Pike</cite>

* Considérez les caractéristique de performance des appels CGO
* Pour réduire les coûts: traitement par lot (ndt: batching)
* Règles pour passer un pointeur entre Go et C
* fichiers syso (détecteur d'accès concurrent, dev.boringssl)

## Techniques avancées

Les techniques spécifique à l'architecture qui exécute le code

* introduction aux caches CPU
  * trappes à performance
  * construire une intuition sur les lignes de cache: taille, bourrage, alignements
  * Outils de l'OS pour obtenir les `cache-miss` (`perf`)
  * `map` vs `slice`
  * composition [SoA ou AoS](https://en.wikipedia.org/wiki/AoS_and_SoA): lecture en colonne ou en ligne; lorsque vous avez un X, avez besoin d'un autre X ou d'un Y ?
  * spétialité temporelle et spatiale: utilisez ce que vous avez et ce qui est proche autant que possible
  * réduisez les résolutions de pointeur
  * préchargement de mémoire explicite; fréquemment ineffectif; lack of intrinsics means function call overhead (removed from runtime)
  * optimisez les 64 premiers bytes de vos structures
* prédictions de branche
  * supprimer les branches du corps des boucles
    `if a { for { } } else { for { } }`
    au lieu de
    `for { if a { } else { } }`
    mesurez la prédiction de branche
    structurez de manière à éviter les branches

    ```go
    if i % 2 == 0 {
        evens++
    } else {
        odds++
    }
    ```

    `counts[i & 1] ++``
    "code sans branchements", mesurez; pas toujours le plus rapide, mais fréquemment plus difficile à lire
    TODO: ASCII class counts example, with benchmarks

* trier vos données peut aider les performances via la localité de cache et la prédictions de branches, même en prenant en compte la durée du tri.
* le surcoût d'appel de fonctions: l'inliner s'améliore
* réduire les copies de données (en incluant les longues listes répétées de paramètres de fonctions)

* Commentaire de Jeff Dean de 2002 sur les nombres (ndt https://gist.github.com/jboner/2841832)
  * les cpus sont devenus plus rapide, mais la mémoire n'a pas suivi

## Concurrence

* Déterminez les pièces qui sont parallélisable et celles qui doivent rester séquentielles
* Les goroutines sont peu coûteuse, pas gratuite
* Optimisation de code multithread
  * faux-partage -> bourrer afin d'ajuster à la taille des lignes de cache
  * vrai-partage -> partitionnement
* Overlap with previous section on caches and false/true sharing
* Synchronisation paresseuse; c'est coûteux, dupliquer le travail peut être plus efficace
* Les paramètres à votre disposition: nombres de worker, taille du lot

Vous devez utiliser un mutex pour protéger un état mutable des accès concurrents.
Si vous avez beaucoup de contention liées à des mutex, vous devez soit réduire les
partage, ou réduire les mutables. Deux manières de réduire le partage sont 1)
partitionner le verrouillage 2) traiter indépendamment puis combiner les résultats.
Pour réduire les mutables: assurez que vos structures de données sont uniquement accédées
en lecture. Vous pouvez aussi réduire la durée nécessaire au verrouillage en
réduisant la section critique -- maintenez le verrou aussi peu que possible. Parfois,
un RWMutex est suffisant, bien qu'ils soient plus lent ils autorisent plusieurs
verrous de lecture.

Si vous partitionnez les verrous, faites attentions aux lignes de caches partagées.
Vous devrez bourrer pour prévenir des rebonds entre processeurs.

var stripe [8]struct{ sync.Mutex; _ [7]uint64 } // mutex is 64-bits; padding fills the rest of the cacheline

N'exécutez rien de coûteux dans la section critique. Ceci inclut les I/O (qui sont peu coûteuse mais lente)

TODO: how to decompose problem for concurrency
TODO: reasons parallel implementation might be slower (communication overhead, best algorithm is sequential, ... )

## Assembly

* De la manière d'écrire du code assembleur pour Go
* Les compilateurs s'améliorent; la barre est haute
* remplacer aussi peu que possible; la maintenance est particulièrement difficile
* bonnes raisons: les instructions SIMD ou d'autres choses qui vont au delà de Go et son compilateur
* C'est très important de mesurez: les améliorations peuvent être énormes (10x pour go-highway)
* zero (go-speck/rc6/farm32), or even slower (no inlining)
* remesurez avec les nouvelles versions pour évaluer la suppression du code assembleur
 * TODO: link to 1.11 patches removing asm code
* always have pure-Go version (purego build tag): testing, arm, gccgo
* brief intro to syntax
* how to type the middle dot
* calling convention: everything is on the stack, followed by the return values.
  - everything is on the stack, followed by the return values
  - this might change https://github.com/golang/go/issues/18597
  - https://science.raphael.poss.name/go-calling-convention-x86-64.html
* using opcodes unsupported by the asm (asm2plan9, but this is getting rarer)
* notes about why inline assembly is hard: https://github.com/golang/go/issues/26891
* all the tooling to make this easier:
  - asmfmt: gofmt for assembly https://github.com/klauspost/asmfmt
  - c2goasm: convert assembly from gcc/clang to goasm https://github.com/minio/c2goasm
   - go2asm: convert go to assembly you can link https://rsc.io/tmp/go2asm
   - peachpy/avo: higher-level assembler in python (peachpy) or Go (avo)
   - differences of above
* https://github.com/golang/go/wiki/AssemblyPolicy
* Design of the Go Assembler: https://talks.golang.org/2016/asm.slide

## Optimiser un service complet

La plupart du temps vous n'aurez pas simplement à optimiser la consommation CPU
d'une unique routine. C'est le cas simple. Si vous avez un service entier à optimiser,
vous devrez regarder le système dans son ensemble. Monitorez. Mesurez. Loggez
pleins de choses dans le temps afin que vous puissiez détecter les dégradations et
mieux comprendre les impacts de vos changements en production.

tip.golang.org/doc/diagnostics.html

* références pour la conception de systèmes : Livre SRE
(ndt: https://landing.google.com/sre/sre-book/toc/), conception d'un système
distribué par la pratique
* outils additionnels : plus de log + analyses
* 2 règles basique : soit vous améliorez ce qui est lent, doit vous l'exécutez moins souvent
* traçage distribué pour chasser les goulets d'étranglements de haut niveau
* conception de requête pour n'attaquer qu'un seul serveur plutôt qu'une grappe
* Vos difficultés de performance pourrait bien ne pas provenir de votre code,
mais vous devrez trouver une solution quoi qu'il en soit
* https://docs.microsoft.com/en-us/azure/architecture/antipatterns/

## Outillage

### Introduction au Profilage

Un petit aide-mémoire pour utiliser les capacités de profilage fournies par `pprof`.
Il existe de nombreux aux autres guide à ce sujet.
Consultez https://github.com/davecheney/high-performance-go-workshop.

TODO(dgryski): videos?

1. Introduction to pprof
   * go tool pprof (and <https://github.com/google/pprof>)
1. Writing and running (micro)benchmarks
   * small, like unit tests
   * profile, extract hot code to benchmark, optimize benchmark, profile.
   * -cpuprofile / -memprofile / -benchmem
   * 0.5 ns/op means it was optimized away -> how to avoid
   * tips for writing good microbenchmarks (remove unnecessary work, but add baselines)
1. How to read it pprof output
1. What are the different pieces of the runtime that show up
  * malloc, gc workers
  * runtime.\_ExternalCode
1. Macro-benchmarks (Profiling in production)
   * larger, like end-to-end tests
   * net/http/pprof, debug muxer
   * because it's sampling, hitting 10 servers at 100hz is the same as hitting 1 server at 1000hz
1. Using -base to look at differences
1. Memory options: -inuse_space, -inuse_objects, -alloc_space, -alloc_objects
1. Profiling in production; localhost+ssh tunnels, auth headers, using curl.
1. How to read flame graphs

### Tracer

### Look at some more interesting/advanced tooling

* other tooling in /x/perf
* perf (perf2pprof)
* intel vtune / amd codexl / apple instruments
* https://godoc.org/github.com/aclements/go-perf

## Appendice: Implémentation de papiers de recherche

Astuce pour implémenter un papier : (Pour les `algorithmes` lisez aussi `structures de données`)

* Ne le faites pas. Commencez par la solution la plus évidente et les structures de données
les plus raisonnable.

Les algorithmes "moderne" tendent vers moins de complexités théorique, mais
un facteur constant cher et des implémentations complexe. L'exemple le plus
classique est l'algorithme du tas de Fibonacci (ndt https://fr.wikipedia.org/wiki/Tas_de_Fibonacci)
. Ils sont significativement plus difficile à produire correctement et ont un
énorme facteur constant. Il y a eu de nombreux papiers publiés pour comparer
plusieurs implémentations de tas avec différentes charge de travail, en général
les d-ary implicite de 4 ou 8 (ndt: http://danielhlarkin.me/pdfs/alenex14-a-back-to-basics-empirical-study-of-priority-queues.pdf)
sont constamment meilleurs. Et même lorsque le tas de Fibonacci devrait être plus
rapide (grâce à sa complexité "decreasekey" de O(1)), les expérimentations de
l'algorithme de parcours en en profondeur (ndt : DFS depth-first search) de
Dijkstra ont de meilleurs résultats avec des algorithmes de tas qui fournissent
l'implémentation les moins complexe.

De la même manière, les abretas ou les listes à enjambement comparés au plus complexe
arbre rouge-noir ou les arbres AVL. Sur du matériel récent, l'algorithme le plus
lent, pourrait être suffisamment rapide, ou même être plus rapide.

> L'algorithme le plus rapide peut fréquemment être remplacé par un autre qui est
presque aussi rapide et bien simple à comprendre.
>
> -- <cite>Douglas W. Jones, Université de l'Iowa</cite>

La complexité ajoutée doit être suffisante pour que le compromis soit justifié.
Un autre exemple concerne les algorithmes d'éviction de cache. Différents
algorithmes peuvent proposées des degrés de complexité plus important pour une
amélioration futile du taux de hit. évidemment, vous ne serez pas capable de
tester tout cela tant que vous n'aurez pas produit une implémentation correcte
et intégrée dans votre système.

De temps en temps le papier présentera des graphiques, non seulement ils auront
tendance à ne publier que les bons chiffres, mais de plus il se pourrait que les
résultats sont faussés pour prouver à quel point le nouvel algorithme est meilleur.

* Choisissez le bon papier
* Recherchez les algorithmes que le papier dit améliorer, et implémenter celui ci.

Fréquemment, les papiers plus anciens seront plus simple à comprendre et fourniront
nécessairement des algorithmes plus simple.

Toutes les études ne sont pas bonne.

Recherchez le contexte d'écriture du papier. Etablissez des hypothèses au sujet
du matériel : espace disque, utilisation mémoire, etc. Des papiers plus anciens
font des compromis différent qui étaient justifiés dans les années 70 ou 80 mais
qui ne sont pas nécessairement bon pour vos cas d'usage. Par exemple, le compromis
entre ce qu'ils appellent un usage mémoire "raisonnable" et l'utilisation de l'espace disque.
L'espace de mémoire disponible aujourd'hui est significativement plus important,
les ssds ont modifiés les pénalités de latence. De la même manière, certains
algorithmes de streaming sont conçus pour du matériel de routeur, ce qui les rend
plus difficile à implémenter dans un logiciel.

Vérifier que les hypothèses du papier s'appliquent à vos données.

Cela prendre du temps de recherche. Vous ne voudrez probablement pas implémenté
le premier papier que vous trouverez.

* Assurez vous de comprendre l'algorithme. Cela peut sembler évident, mais ce sera
impossible à débugger autrement.

  <https://blizzard.cs.uwaterloo.ca/keshav/home/Papers/data/07/paper-reading.pdf>

  Une bonne compréhension peut vous permettre d'extraire l'idée clé du papier
  et éventuellement n'appliquer que cela à votre problème, plutôt que de ré-implémenter
  l'intégralité de la chose.

* Le papier original d'une structure de données ou d'un algorithme n'est pas toujours le meilleur.
D'autres papiers plus récent peuvent fournir de meilleurs explications.

* Certains papiers référence du code source que vous pourrez consulter, cependant
  1) le code académique est presque universellement horrible
  2) prenez connaissance des restrictions légale ("research purposes only")
  3) prenez garde aux bugs; cas particulier, vérification d'erreur, performance etc.

  Recherchez aussi des implémentations sur Github : Ils pourraient avoir les mêmes (ou pas)
  bugs que vôtre implémentation.

Pour aller plus loin sur ce sujet:
* <https://www.youtube.com/watch?v=8eRx5Wo3xYA>
* <http://codecapsule.com/2012/01/18/how-to-implement-a-paper/>
