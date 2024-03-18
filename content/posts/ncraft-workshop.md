+++
author = "Mickael Metesreau"
title = "NCraft Workshop"
date = "2015-08-11"
description = "NCraft Workshop"
tags = []
+++

Le 21 mai dernier, en ouverture de la [NCrafts](http://ncrafts.io/), se tenait une série de workshops. J'ai profité de l'occasion pour assister à la session **Crafting code** donnée par [Sandro Mancuso](https://twitter.com/sandromancuso). Le contenu était une version allégé d'un programme fait pour tenir 3 jours laissant la part belle à la pratique. La journée fut donc chargée avec un gros focus sur des exercices en pair. Voici mes retours : 

### Le nommage de variable

On entend souvent qu'il n'y a que deux choses difficiles en informatique : l'invalidation de cache et le nommage. La première réflexion qui nous a été soumise concernait notre propension à appeler un chat un chien. 

Voyez les exemples ci-dessous :

``` shell
BankAccount sut
BankAccount ba
BankAccount test
```

Dans un objectif de lisibilité et de compréhension, pourquoi ne pas tout simplement nommer la variable bankaccount ?

### Convention sur les classes de test

Pour la suite des exercices, Sandro Mancuso nous a proposé d'utiliser sa convention pour l'écriture des tests. Il en existe d'autres mais j'avoue que j'aime bien cette façon de construire ses tests :

``` csharp
class MyClassShould
{
  void Do_Something() {}
}
```

### Étapes pour créer un test

Ecrire des tests? Oui, mais par où commencer?

``` csharp
class MyClassShould (1)
{
  void Do_Something() (2)
  {
    // Arrange, Given ................ (5)
    // Act,     When  ................ (4)
    // Assert,  Then  ................ (3)
  }
}
```

Il semble logique de commencer par écrire nos assertions car cela nous permet de poser les bases de ce que l'on souhaite tester.

A la question, est-il préférable d'avoir une seule assertion dans un test unitaire, la réponse fut "tout dépend du contexte" (Oui je sais, c'est bien une réponse d'informaticien :p). En fait, si une méthode engendre plusieurs mutations d'états alors il semble logique de pouvoir tous les vérifier.

### Classic TDD

Le premier exercice était le [kata Roman Number](http://agilekatas.co.uk/katas/romannumerals-kata). Nous devions travailler sur une solution en adoptant une stratégie Classic TDD (bottom-up). Voici les conclusions de fin d'exercice :

- Oublier les cas spécifiques dans un premier temps (4, 9, etc)
- Ne pas créer de test pour des cas qui ne font pas avancer le code
- Une fois l'algorithme gérant la majorité des cas écrit, on peut passer aux exceptions. Le nombre d'exception étant ici fini, la proposition est de les traiter comme des cas normaux.

### Double Loop TDD

Après cette séance pratique, retour à la théorie avec une introduction à la double loop TDD : le principe est d'intégrer la première boucle de feedback dans une seconde de plus haut niveau. Cela consiste à ajouter un test d'acceptance qui échoue avant d'implémenter notre fonctionnalité à travers des itérations TDD.

![Image](/images/posts/ncraft-workshop/doublelooptdd.jpg)

### Outside-In TDD

Le second exercice nous a poussé vers la tendance Mockist TDD que je n'avais jamais l'occasion de tester. Par concept, on a débuté par un test d'acceptance pour enchaîner avec des tests unitaires (Oh la belle double loop tdd). 

Le fait de commencer par implémenter un test fonctionnel et de travailler sur un exemple orienté métier nous a réellement incité à utiliser des mocks dès le début et donc à faire des choix de design très tôt. Même s'il y a toujours une phase de refactoring, des partis pris sont rapidement posés et cela nécessite une certaine expérience pour bâtir quelque chose de propre. Avec cette approche le design n'est plus vraiment émergent mais je trouve particulièrement intéressant de voir les différents agencements possibles et les architectures vers lesquels ils mènent.

Pour un comparatif entre les deux écoles, vous pouvez lire l'article [suivant](http://codurance.com/2015/05/12/does-tdd-lead-to-good-design/). 

### Legacy code

La dernière partie du workshop se concentrait sur la gestion d'un code hérité. Avant d'effectuer une quelconque modification, notre premier travail est de comprendre le code. Une fois cette étape franchie, on peut commencer à couvrir notre code avant de se lancer de le refactoring. Mais par où commencer ? 

Le conseil est de faire une analyse visuelle de la structure du code :

```
{
  ............
    .............
      ............... Tester le retour le plus court
    .............

  ........
    ...........
      .............
        ................
          ................. Refactorer la branche la plus fine
    ...........
}
```

Enfin ne pas oublier d'utiliser un outil de code coverage ;)

### Comment gérer les dépendances statiques ?

Une solution en trois étapes a été proposée pour gérer le cas de dépendances statiques comme celui des singletons afin de pouvoir commencer à couvrir du code :

- Isoler la dépendance dans une méthode de notre classe 
- Marquer cette classe comme overridable
- Créer une classe héritée qui surcharge le comportement de notre dépendance

Une fois notre code couvert par un test, on peut commencer à remplacer nos dépendances statiques par de l'injection de dépendances. Si vous souhaitez explorer plus en détail cette partie, une explication détaillée a été écrite sur le blog de [Xebia](http://blog.xebia.fr/2015/01/23/legacy-code-se-defaire-des-dependances-statiques/).

### Comment améliorer la lisibilité de notre code ?

Unifier les niveaux d'abstraction afin d'éviter de jongler entre termes métier et termes techniques.

Exemple :

``` csharp
List<string> GetTrips()
{
  var trips = service.GetTrips(user);
  return trips ?? new List<string>(); 
}  
```

Refactorisation :

``` csharp
const List<string> noTrip = new List<string>();

List<string> GetTrips()
{
  var trips = service.GetTrips(user);
  return trips ?? noTrip;
}    
```

Extraire les initialisations et les configurations d'objets dans des builders pour gagner en lisibilité.

Exemple :

``` csharp
List<Friend> GetFriends(User user)
{
  var friend = new Friend();
  friend.LastName = "Smith";
  friend.FirsName = "John";
  user.AddFriend(friend);
  return user.Friends;
}
```

Refactorisation :

``` csharp
List<Friend> GetFriends(User user)
{
  var friend = FriendBuilder
                 .NewFriend()
                 .WithLastName("Smith")
                 .WithFirstName("John");

  user.AddFriend(friend);
  return user.Friends;
}
```

### Impressions générales

La journée fut un bon rappel et m'a permis de découvrir plein de nouvelles choses. Il est toujours intéressant de travailler à deux afin d'échanger sur des manières différentes d'aborder un problème. Sandro Mancuso est une personne passionné et pleine d'énergie, c'est toujours une chance de pouvoir assister à des workshops avec des speakers de ce niveau. N'hésitez pas à jeter un coup d’œil sur l'ensemble des solutions des exercices sur [Youtube](https://www.youtube.com/channel/UCacyhBPMQpC4Vi-WqtrRpBw). J'ai aussi mis sur [Github](https://github.com/mmetesreau/ncrafts-workshop) le code produit durant cette journée. Enfin si vous avez raté la NCraft, sachez que les vidéos sont disponibles [ici](videos.ncrafts.io).