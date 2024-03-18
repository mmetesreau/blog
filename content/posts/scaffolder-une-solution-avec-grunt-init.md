+++
author = "Mickael Metesreau"
title = "Scaffolder une solution avec grunt init"
date = "2015-01-26"
description = "Scaffolder une solution avec grunt init"
tags = []
+++

Envie de démarrer un projet et même de faire un peu de TDD ? Mais avant tout, il faut configurer une nouvelle solution. [Grunt init](http://gruntjs.com/project-scaffolding) est un outil permettant d'automatiser la création de vos projets. Il va tout simplement nous permettre de créer une arborescence à partir de templates et de réponses à quelques questions.

Basé sur le [projet](https://github.com/nosami/grunt-init-csharpsolution) de Jason Imison, amusons nous à bootstrapper une solution CSharp avec des tests [NUnit](http://www.nunit.org/).

### Prérequis 

Pour suivre la suite de cet article, vous avez besoin d'installer [node](http://nodejs.org/) pour faire tourner grunt-init.

### HelloWorld 

Installons grunt-init avec le gestionnaire de package node :

``` bash
npm install -g grunt-init
```

Le but de ce premier exemple est de créer un simple fichier. Celui-ci contiendra une chaine de caractères demandée lors de la génération.

Sur Windows, grunt-init cherche nos templates dans le dossier c:\Users\\[username]\\.grunt-init. Nous allons y ajouter l'arborescence suivante :

```
helloworld
  |---> template.js
  |---> rename.json
  |---> root 
          |---> hello.txt
```

Le fichier template.js liste l'ensemble des actions à effectuer. Dans notre cas, nous demandons une valeur à l'utilisateur avant de lancer la copie et le traitement :

``` js
exports.template = function(grunt, init, done) {
  init.process({}, [
    init.prompt('name')
  ], 
  function(err, props) {
    var files = init.filesToCopy(props);

    init.copyAndProcess(files, props);

    done();
  });
};
```

Sachez que l'instruction init.prompt('name') permet d'invoquer une des questions pré-configurées de grunt-init.

Le dossier root fourni l'ensemble des fichiers à copier. Dans notre cas, on y trouve le fichier hello.txt contenant un placeholder qui sera remplacé lors du traitement :

```
Hello {%= name %}
```

Enfin le fichier rename.json permet de configurer le mapping entre les fichiers sources et les fichiers de destination. C'est ici que nous pouvons exclure ou renommer des fichiers. Pour notre exemple, nous n'avons pas de besoin spécifique :

``` js
{}
```

Notre premier template est maintenant fini. Pour l'utiliser, lançons la commande suivante depuis un terminal :

``` bash
grunt-init helloworld 
``` 

Vous devriez obtenir ceci :

![screenshot3](/images/posts/scaffolder-une-solution-avec-grunt-init/screenshot3.gif)


### CSharp template 

Créons la structure de notre template de solution csharp :

```
csharp
  |---> template.js
  |---> rename.json
  |---> root 
```

Le dossier root va contenir les fichiers suivants :

```
root
  |---> name.sln
  |---> name
          |---> name.csproj
          |---> Properties
                  |---> AssemblyInfo.cs
  |---> name.Tests
          |---> name.Tests.csproj
          |---> Tests.cs
          |---> packages.config
          |---> Properties
                  |---> AssemblyInfo.cs
  |---> .nuget 
          |---> NuGet.config
          |---> NuGet.targets
          |---> NuGet.exe
```

Pour obtenir ce template, je suis parti d'une solution créée avec Visual Studio contenant un projet de test NUnit utilisant [NFluent](http://www.n-fluent.net/). 

Afin de rendre le tout générique, j'ai ajouté les placeholders **{&#37;= name &#37;}**, **{&#37;= projectGuid &#37;}** et **{&#37;= testProjectGuid &#37;}** dans les fichiers suivants :

- Tests.cs 
- name.sln 
- name.csproj 
- name.Tests.csproj

Cela nous permet de générer des namespaces et des références corrects. Petit exemple avec le fichier Tests.cs :

``` csharp
using NUnit.Framework;
using NFluent;

namespace{%=name %}.Tests
{
  [TestFixture]
  publicclassTests
  {
    [Test]
    public void Should_be_true()
    {
      var heroes="Batman and Robin";
      Check.That(heroes)
         .Not.Contains("Joker")
         .And.StartsWith("Bat")
         .And.Contains("Robin");
    }
  }
}
``` 

Le fichier template.js va se contenter d'effectuer le traitement en excluant le binaire NuGet.exe :

``` js
exports.template = function(grunt, init, done) {
  function generateUUID() {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
      var r = Math.random() * 16 | 0,
      v = c == 'x' ? r : (r & 0x3 | 0x8);
      return v.toString(16);
    }).toUpperCase();
  }

  init.process({}, [
    init.prompt('name')
  ], 
  function(err, props) {
    props.projectGuid = generateUUID();
    props.testProjectGuid = generateUUID();

    var files = init.filesToCopy(props);

    init.copyAndProcess(files, props, {noProcess: '.nuget/**'});

    done();
  });
};
```

Il nous reste à effectuer le mapping de nos fichiers de destination afin de prendre en compte le nom fourni en paramètre par l'utilisateur. Le plus simple est de remplacer la chaîne "name" de tous les fichiers/dossiers par celle indiquée par l'utilisateur après la copie des fichiers.


``` js
//...
var files = init.filesToCopy(props);

for (var file in files) {
  var path = files[file];
  var newFile = file.replace(/name/g, props.name);
  if (file !== newFile) {
    files[newFile] = path;
    delete files[file];
  }
}

init.copyAndProcess(files, props, {noProcess: '.nuget/**'});

done();
```

Vous pouvez enfin générer une solution prête à l'emploi en une seule commande :)

``` bash
grunt-init csharp 
```