+++
author = "Mickael Metesreau"
title = "Intégration continue avec Travis"
date = "2014-10-07"
description = "Intégration continue avec Travis"
tags = []
+++

L'intégration continue et le déploiement sont des sujets capitaux dans le développement logiciel. De nombreuses offres se sont structurées autour de ces besoins comme [appveyor](http://www.appveyor.com/) ou [codeship](https://codeship.io).

Dernièrement, j'ai utilisé [Travis](https://travis-ci.org) pour automatiser le build, le test et le déploiement d'une application Mono sur Heroku. Je vous en propose un petit tour d'horizon :)

### Une bonne solution ?

Travis est pratique mais n'est pas adapté à tous les besoins :

- (+) Gratuit pour les projets Open Source
- (+) Support des langages majeurs Java, Js, PHP, Python, Ruby, Mono...
- (+) Support du déploiement
- (-) Fortement lié à Github
- (-) Support des dépôts privés assez chers
- (-) Pas de rapport spécifique pour les tests
- (-) Pas d'intégration avec EC2

### Exemple pour une application Mono

Toute la configuration passe par le fichier .travis.yml à la racine de votre projet.

Configuration des pré-requis de compilation et de test :

``` bash
before_install:
- sudo apt-get update -qq > /dev/null
- sudo apt-get install -qq mono-devel mono-gmcs nunit-console > /dev/null
- mozroots —import --sync
- mv -f .nuget/NuGet.mono.targets .nuget/NuGet.targets
- export EnableNuGetPackageRestore=true
```

Configuration du build et des tests :

``` bash
scripts:
- xbuild HelloWorld.sln
- nunit-console HelloWorld.Tests/bin/Debug/HelloWorld.Tests.dll
```

Configuration du déploiement pour Heroku :

``` bash
deploy:
- provider: heroku
- buildpack: https://github.com/friism/heroku-buildpack-mono.git
- api_key: "app-key"
- app: "app-name"
```

Après avoir configuré votre dépôt sur Travis, chaque push va déclencher un build. 

![Image](/images/posts/integration-continue-avec-travis/image1-1.PNG)

Il est quand même possible de restreindre le déclenchement à certaines branches :

``` bash
branches:
only:
- master
```

Enfin si l'intégration échoue à une étape, vous êtes notifié par email.

Le projet d'exemple est disponible sur [Github](https://github.com/mmetesreau/nancyfx-travisci-heroku).