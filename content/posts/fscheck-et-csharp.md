+++
author = "Mickael Metesreau"
title = "Tester ces spécifications avec FsCheck et C#"
date = "2015-06-18"
description = "Tester ces spécifications avec FsCheck et C#"
tags = []
+++

Vous êtes fatigué de chercher des cas de test ? Vous ne connaissez pas vos cas limites ? Alors vous devriez jeter un coup d'oeil à [FsCheck](https://github.com/fscheck/FsCheck). Cet outil se charge de tester des propriétés pour un large nombre de cas automatiquement générés. Oui j'ai dit propriété mais ne vous inquiétez pas :). Dans la terminologie FsCheck une propriété est simplement une spécification que doit satisfaire votre code.

Et comme rien ne vaut mieux qu'un exemple, considérons l'implémentation suivante pour le kata [FizzBuzz](http://www.codingdojo.org/cgi-bin/index.pl?KataFizzBuzz) :

``` csharp
public static string DoFizzBuzz(int i)
{
    bool fizz = i % 3 == 0;
    bool buzz = i % 5 == 0;
    if (fizz && buzz)
      return "FizzBuzz";
    else if (fizz)
      return "Fizz";
    else if (buzz)
      return "Buzz";
    else
      return i.ToString();
}

```

Et les tests qui vont avec (avec [NUnit](http://www.nunit.org/)) :

```csharp
[TestCase(2, Result = "2")]
[TestCase(3, Result = "Fizz")]
[TestCase(5, Result = "Buzz")]
[TestCase(15, Result = "FizzBuzz")]
public string Given_Value_It_Must_Return_The_Given_Result(int value)
{
  return FizzBuzzer.DoFizzBuzz(value);
}
```

Nous avons maintenant une solution qui semble marcher :). Cependant nous ne pouvons pas garantir qu'elle fonctionne avec l'ensemble complet des nombres entiers de 1 à 100. 

Pour résoudre ce problème, la première idée qui vient à l'esprit est de tester l'ensemble des cas de 1 à 100. Cela est ici possible car le nombre de cas n'est pas très grand et est finie. Mais l'idée ne semble pas viable pour d'autres exemples. 

Un compromis serait de tester une sous partie de cet ensemble et même de l'automatiser. C'est maintenant que FsCheck fait son entrée.

Après avoir ajouté le package [nuget](https://www.nuget.org/packages/FsCheck/), nous allons créer un [générateur](https://fscheck.github.io/FsCheck/TestData.html) qui nous permettra de choisir une valeur aléatoire dans l'intervalle des entiers de 1 à 100.

``` csharp
private static Gen<int> DivisibleBy(int divisor, int? except = null)
{
  return from number in Gen.Choose(0,100)
         where number % divisor == 0
         where !except.HasValue || number % except.Value != 0
         select number;
}
```

Nous ajoutons la possibilité d'exclure un diviseur pour éviter des erreurs lors de la génération d'ensemble pour le Fizz et le Buzz.

Dans un second temps, créons un nouveau test qui va utiliser ce générateur pour tester la propriété Fizz de notre méthode DoFizzBuzz.

``` csharp
[Test]
public void Given_A_Number_Divisible_By_3_It_Must_Contain_Fizz()
{
  var arb = Arb.From<int>(DivisibleBy(3,5));
  Prop.ForAll<int>(arb, i => FizzBuzzer.DoFizzBuzz(i) == "Fizz")
    .QuickCheckThrowOnFailure();
}
```

Et voilà, en deux lignes, nous avons écrit un test qui va essayer de trouver automatiquement une valeur contredisant une de nos spécifications.

![Image](/images/posts/fscheck-et-csharp/capture.png)

Revenons un peu sur l'api de FsCheck :

- Arb: Permet de créer un objet contenant un générateur et un shrinker. Le shrinker est optionnel mais autorise à contrôler la manière de réduire un ensemble. En effet, FsCheck va toujours chercher à vous retourner le plus petit ensemble invalidant votre propriété.

- Prop : Tout simplement le point d'entrée permettant de créer une propriété.

- QuickCheckThrowOnFailure : Il existe plusieurs moyens d'intégrer FsCheck dans nos runners de test. Le premier consiste à utiliser un addin et à modifier nos tests pour qu'il retourne une propriété. Un exemple est disponible [ici](https://github.com/fscheck/FsCheck/tree/master/examples/FsCheck.NUnit.CSharpExamples). La seconde option est d'utiliser les méthodes d'extension QuickCheck et QuickCheckThrowOnFailure qui vont jouer différents cas pour une propriété directement dans le test. Il faut cependant faire attention avec QuickCheck car contrairement à QuickCheckThrowOnFailure, le test n'échouera jamais...

Pour finir, nous pouvons généraliser le test précédent pour nos autres cas :

``` csharp
[TestCase(3, "Fizz", 5)]
[TestCase(5, "Buzz", 3)]
[TestCase(15, "FizzBuzz", null)]
public void Given_A_Number_Divisible_By_Divisor_It_Must_Contain_Expected(int divisor, string expected,  int? except = null)
{
  var arb = Arb.From<int>(DivisibleBy(divisor));
  Prop.ForAll<int>(arb, i => FizzBuzzer.DoFizzBuzz(i) == expected)
    .QuickCheckThrowOnFailure();
}
```

Nos tests passent ! Et nous sommes assurés de tester de manière plus complète notre implémentation grâce à FsCheck.

![Image](/images/posts/fscheck-et-csharp/capture1.png)

Bien entendu, pour plus d'informations, vous pouvez vous référer à la [documentation](https://fscheck.github.io/FsCheck/) :) 
