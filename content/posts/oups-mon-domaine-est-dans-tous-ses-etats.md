+++
author = "Mickael Metesreau"
title = "Oups mon domaine est dans tous ses états"
date = "2015-11-13"
description = "Oups mon domaine est dans tous ses états"
tags = []
+++

Les techniques de Property-Based Testing introduisent généralement de l’aléatoire afin de générer de nombreuses entrées et de tenter de trouver des contre exemples aux faits que nous posons dans nos tests. Il est donc souvent nécessaire de créer des générateurs pour nos entrées métier. Et ces générateurs doivent être non seulement capables de construire nos objets mais aussi d'assurer la validité de ceux-ci. 

Imaginez la modélisation d'une personne avec un âge. Il n'est pas possible de se contenter de représenter cet age par un champ entier : il doit être positif.

Une des solutions pour résoudre cette problématique est de tout simplement rendre impossible la représentation d'états invalides. Comment ? En utilisant le système de types.

Pour la suite de cet article, je vais reprendre l'exemple que Mark Seemann ([@ploeh](https://twitter.com/ploeh?lang=en)) expose dans ce [talk](https://vimeo.com/144800642).

## Notre domaine

Pour faire de la modélisation, il nous faut un domaine. Nous allons utiliser ici les règles du [kata tennis](http://codingdojo.org/cgi-bin/index.pl?KataTennis) :

- Un match de tennis voit s'affronter **deux joueurs**
- Dans un jeu, les scores possibles sont **Love(0)**, **15**, **30** et **40** 
- Si un joueur est à 40 et gagne le point alors il gagne le **jeu**

C'est relativement simple mais il ne faut pas oublier quelques règles spéciales :

- Si les deux joueurs sont à 40 alors le score est **Deuce**
- Si le score est Deuce, le joueur vainqueur du point a un **Avantage**
- Si le joueur avec un Avantage gagne un point alors il gagne le **jeu** sinon le score est de nouveau Deuce

## Une implémentation

Pour commencer, représentons la notion de joueur. L'ensemble des joueurs est fini et contient deux éléments :

``` fsharp
type Player = PlayerOne | PlayerTwo
```

Oui c'est du F#. Je reviendrais sur le pourquoi plus tard. Si vous ne connaissez pas F#, alors sachez que je viens de créer une [union discriminée](https://msdn.microsoft.com/fr-fr/library/dd233226.aspx). Cela  ressemble de loin à une énumération C# et permet de modéliser mes deux états au sein d'un type.

Passons maintenant à la notion de point. Une première solution consiste à créer un type alias vers int.

``` fsharp
type Point = int
```

Il est cependant facile de voir que ce choix n'est pas judicieux. La majorité des éléments de l'ensemble int sont des états invalides dans notre domaine métier. Par exemple, -10, 3, 206 ne sont pas des valeurs acceptables. 

Tentons notre chance avec une seconde union discriminée qui nous permet de restreindre les valeurs possibles : 

``` fsharp
type Point = Love | Fifteen | Thirty | Forty
```

Il est aussi à noter que cette solution nous permet de réutiliser directement le langage métier dans notre code.

Créons maintenant une structure nous permettant de stocker le score d'une partie entre deux joueurs :

``` fsharp
type PointsData = { 
  PlayerOnePoint: Point 
  PlayerTwoPoint: Point 
}
```

Nous avons réduit les cas d'erreurs mais il reste possible de créer un état invalide :

``` fsharp
{PlayerOnePoint = Forty; PlayerTwoPoint = Forty} 
```

En effet, lorsque les deux joueurs sont à Forty alors le score devrait être Deuce. Modifions donc Point en retirant Forty et créons un nouveau type  qui modélisera l'ensemble des cas (40-0, 40-15, 40-30, 30-40, 15-40, 0-40) :

``` fsharp
type Point = Love | Fifteen | Thirty

type PointsData = { 
  PlayerOnePoint: Point 
  PlayerTwoPoint: Point 
}

type FortyData = { 
  Player : Player 
  OtherPlayerPoint : Point
}
```

Dans le cas où un des joueurs est à Forty et l'autre non, nous avons représenté l'état du jeu par un type plutôt que par un champ. En suivant cette  stratégie, nous devons seulement stocker le type du joueur ayant atteint Forty et le nombre de points de son adversaire.

Maintenant que nous avons à notre disposition la modélisation par des types de deux des états possibles d'un score, nous allons pouvoir nous servir de la puissance des unions discriminées :

``` fsharp
type Score = Points of PointsData | Forty of FortyData
```

Ce qui est génial avec les unions discriminées, c'est qu'il est possible de spécifier un ensemble composé d’éléments utilisant des valeurs différentes.
 
Continuons avec le cas particulier Deuce. Celui-ci ne porte pas d'information particulière. Ajoutons le directement à notre union discriminée :

``` fsharp
type Score = Points of PointsData | Forty of FortyData | Deuce
```

Il nous reste à traiter les cas Advantage et Game. Ces état sont étant liés à un joueur, nous pouvons compléter notre type score pour offrir tous les cas possibles de la manière suivante :

``` fsharp
type Score = Points of PointsData | Forty of FortyData | Deuce | Advantage of Player | Game of Player
```

Nous avons terminé, il est désormais impossible d'instancier un score qui serait invalide d'un point de vue business.

## Pourquoi F# ?

Regardons le code complet :

``` fsharp
type Player = PlayerOne | PlayerTwo
type Point = Love | Fifteen | Thirty

type PointsData = { 
  PlayerOnePoint: Point 
  PlayerTwoPoint: Point 
}

type FortyData = { 
  Player : Player 
  OtherPlayerPoint : Point
}

type Score = Points of PointsData | Forty of FortyData | Deuce | Advantage of Player | Game of Player
```

Que ce soit en terme de lignes écrites ou de concision, je ne pense pas qu'il soit possible de faire mieux en C#. Et puis plus le code est court et lisible, plus la maintenance est facile non ? Etant donnée la complexité et l'importance du métier (c'est quand même ce qui nous fait manger), pourquoi devrions nous nous priver d'un outil de modélisation si  puissant ? Qui plus est disponible sur la plateforme .NET.

Si vous cherchez d'autres raisons, je vous conseille de faire un tour sur le blog **F# for fun et profit** et de jeter un coup d’œil à cet excellent [article](http://fsharpforfunandprofit.com/series/why-use-fsharp.html). 

Have fun !