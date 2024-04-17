+++
author = "Mickael Metesreau"
title = ".NET + Node = Edge.js"
date = "2015-08-24"
description = ".NET + Node = Edge.js"
tags = []
+++

[Edge.js](http://tjanczuk.github.io/edge/) est disponible sur Windows, Linux et Mac OS avec .NET 4.5 ou Mono 3.4 et permet d'exécuter du code .NET in-process depuis une application [Node](https://nodejs.org/). Ce package fonctionne avec les langages .NET (C#, F#, ...) mais aussi Powershell, T-SQL, Python, etc.

![Image](/images/posts/edgejs/dotnetnodebridge.png)

Voici quelques cas d'utilisation :

- Intégrer des composants .NET existant
- Utiliser ADO.NET
- Faire du multi threading avec Node
- Créer des extensions Node en C# 

## Hello World

Initialisons notre application depuis notre terminal :

``` bash
$>npm init
$>npm install --save edge
```

Créons notre fichier hello.js :

``` javascript
var edge = require('edge');

var helloWorld = edge.func(function () {/*
  async (input) => { 
    return ".NET Welcomes " + input.ToString(); 
  }
*/});

helloWorld('JavaScript', function (error, result) {
  if (error) throw error;
  console.log(result);
});
```

Lançons le tout :

``` bash
$>node hello.js
.NET Welcomes JavaScript
```

Magique non ? En fait pas vraiment, Edge.js se charge de marshal les données entre la CLR et [V8](https://en.wikipedia.org/wiki/V8_(JavaScript_engine)) : 

![Image](/images/posts/edgejs/interop.png)

## Simple-Nunit-Runner

Nous allons maintenant créer un reporter [NUnit](http://www.nunit.org/) qui nous permettra de récupérer le résultat de nos tests dans Node.

Dans un premier temps, nous allons créer un listener tout simple en C# pour les TestRunners de NUnit. Pour cela, nous allons créer une classe qui héritera de ```EventListener``` (disponible dans [NUnit.Core](https://www.nuget.org/packages/NUnit.Core/)). Fonctionnellement, nous nous contenterons de stocker les résultats des tests.

``` csharp
public class SimpleListener : EventListener
{
  public List<TestResult> TestResults { get; set; }

  public SimpleListener()
  {
    TestResults = new List<TestResult>();
  }

  public void TestFinished(TestResult result)
  {
    TestResults.Add(result);
  }
  
  // Not implemented
  //...
}
```

Écrivons maintenant notre point d'entrée :

``` csharp
var listener = new SimpleListener();
var runner = new RemoteTestRunner();

runner.Load(new TestPackage("tests", new[] { (string)input }));
runner.Run(listener, TestFilter.Empty, false, LoggingThreshold.Off);

Func<TestResult, object> mapper = (res) => new {
  name = tr.Name,
  description = tr.Description,
  message = tr.Message,
  time = tr.Time
};

return new {
  completed = listener.TestResults.Where(tr => tr.IsSuccess).Select(mapper),
  errors = listener.TestResults.Where(tr => !tr.IsSuccess).Select(mapper)
};
``` 

La méthode ```Invoke``` se charge de lancer un runner avec notre listener sur la dll passée en paramètre puis de formater les sorties des tests dans de nouveaux objets.

Au final cela nous donne :

``` csharp
var edge = require('edge'),
	    path = require('path');

var reporter = edge.func({
  source: function () {/*
    using NUnit.Core;
	    using System;
	    using System.Collections.Generic;
	    using System.Linq;
	    using System.Threading.Tasks;
	    
	    public class Startup
	    {
      			public async Task<object> Invoke(object dllPath)
      			{
			        var listener = new SimpleListener();
			        var runner = new RemoteTestRunner();
			        runner.Load(new TestPackage("tests", new[] { (string)dllPath }));
			        runner.Run(listener, TestFilter.Empty, false, LoggingThreshold.Off);

			        return new {
			          completed = listener.TestResults.Where(tr => tr.IsSuccess)
			                          .Select(tr => new {
			                            name = tr.Name,
			                            description = tr.Description,
			                            message = tr.Message,
			                            time = tr.Time
			                          }),
			          errors = listener.TestResults.Where(tr => !tr.IsSuccess)
			                       .Select(tr => new {
			                         name = tr.Name,
			                         description = tr.Description,
			                         message = tr.Message,
			                         time = tr.Time
			                       }),
			         };
      			}
	    }

		    public class SimpleListener : EventListener
    		{
		      public List<TestResult> TestResults { get; set; }

      public SimpleListener()
      {
		        TestResults = new List<TestResult>();
      }

		      public void TestFinished(TestResult result)
		      {
		        TestResults.Add(result);
		      }

		      public void RunFinished(Exception exception) { }
		      public void RunFinished(TestResult result) { }
		      public void RunStarted(string name, int testCount) { }
		      public void SuiteFinished(TestResult result) { }
		      public void SuiteStarted(TestName testName) { }
		      public void TestOutput(TestOutput testOutput) { }
		      public void TestStarted(TestName testName){ }
		      public void UnhandledException(Exception exception) { }
    }
  */},
	  references: [
		    path.join(__dirname, "lib/nunit.core.dll"),
	    path.join(__dirname, "lib/nunit.core.interfaces.dll"),
	    path.join(__dirname, "lib/nunit.framework.dll")
  	]
});
reporter('C:\\path\\to\\tests.dll', function (error, result) {
  if (error) throw error;
  console.log(result);
});
```

Un peu long, mais on touche au but :). Vous avez peut être remarqué que le code a été légèrement modifié ? 

- La classe ```Startup``` : comme nous embarquons deux classes dans notre code C#, il n'est pas possible d'utiliser une méthode anonyme comme point d'entrée.
- L'appel à ```edge.func``` : il est nécessaire d'indiquer à edge.js le chemin des dépendances à charger pour compiler le code. 

Voici un exemple de sortie obtenu avec le code précédant :

![Image](/images/posts/edgejs/capture.png)

## Mais pourquoi faire ?

Node propose de nombreux task runners et pléthore d'extensions. Une des possibilités qui m'est venu à l'esprit est d'encapsuler cette logique dans un plugin [Grunt](http://gruntjs.com/). Il est ensuite possible d'utiliser [grunt-contrib-watch](https://www.npmjs.com/package/grunt-contrib-watch) et [grunt-msbuild](https://www.npmjs.com/package/grunt-msbuild) afin de créer un outil simple de Continous Testing.

![Image](/images/posts/edgejs/687474703a2f2f63646e2e676f6561742e66722f696d672f6772756e746e756e6974636c692e706e67.png)

Si vous voulez jeter un coup d’œil aux sources, le tout est disponible [ici](https://github.com/mmetesreau/grunt-simple-nunit-runner) et [ici](https://github.com/mmetesreau/sample-grunt-simple-nunit-runner).

De plus, vous avez peut être remarqué que l'écosystème .NET s'ouvre de plus en plus. On peut voir des outils apparaître pour Vim, Sublime Text, Atom, Bracket sur tous les systèmes d'exploitation. La communauté semble très active sur Atom et celui-ci se base sur [electron](https://github.com/atom/electron) qui n'est grosso modo qu'un runtime Node. Il devient alors facile de créer des nouvelles extensions et de nouveaux ponts pour cet IDE. 

A titre d'exemple, voici le résultat d'un de mes POCs parsant la sortie d'[OpenCoverage](https://github.com/OpenCover/opencover) avec [ReportGenerator](https://github.com/danielpalme/ReportGenerator) et surlignant les chemins non couverts.

![Image](/images/posts/edgejs/capture.gif)
