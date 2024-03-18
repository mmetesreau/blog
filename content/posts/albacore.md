+++
author = "Mickael Metesreau"
title = "Exotique .NET - Rake et Albacore"
date = "2015-06-22"
description = "Exotique .NET - Rake et Albacore"
tags = []
+++

Les tasks runners sont des applications bien pratiques permettant d’exécuter un ensemble d'actions comme compiler un projet, déployer un site web ou lancer une migration de base de données. Vous vous demandez peut être pourquoi utiliser un programme pour effectuer ces opérations alors que nous disposons tous d'IDE où la magie du clic est omniprésente ? Et bien tout simplement pour automatiser et gérer les dépendances entre ces actions. Moins nous avons de travail à faire pour effectuer des tâches répétitives, mieux nous nous portons non ?

Voici un tour d'horizon de Rake et de comment il peut faciliter notre vie de développeur .NET.  

### Rake

[Rake](https://github.com/ruby/rake) est un programme Ruby similaire à [Make](https://fr.wikipedia.org/wiki/Make). Il se base sur les fichiers Rakefile qui contiennent des tâches. En voici un exemple :

``` ruby
task :t1 do
  puts "Exécution de la tâche 1"
end  

task :t2 do
  puts "Exécution de la tâche 2"
end
```

Une fois [ruby](https://www.ruby-lang.org/en/downloads/), [gem](https://rubygems.org/pages/download) puis [rake](https://github.com/ruby/rake#installation) installés, le lancement de la commande rake dans le répertoire contenant le fichier Rakefile donnera ceci :

``` bash
$>gem install rake
$>rake t1
Exécution de la tâche  1
$>rake t2
Exécution de la tâche 2
```

Assez logiquement, les symboles t1 et t2 lancent respectivement les tâches 1 et 2. Rajoutons maintenant une dépendance entre la tâche t1 et t2 :

``` ruby
task :t1 do
  puts "Exécution de la tâche 1"
end  

task :t2 => :t1 do
  puts "Exécution de la tâche 2"
end
```

Après cette modification, le fait de lancer la tâche t2 exécutera, dans l'ordre, t1 puis t2 :

``` bash
$>rake t1
Exécution de la tâche 1
Exécution de la tâche 2
```

### Albacore

[Albacore](https://github.com/Albacore/albacore) est un ensemble de tâches rake visant à faciliter les builds .NET comme par exemple créer un package nuget, effectuer un build ou lancer des tests.

Installons tout d'abord Albacore :

``` bash
$>gem install albacore 
```

Postulons posséder une solution ```AlbacoreBuild.sln``` contenant deux projets ```AlbacoreBuild.csproj``` et ```AlbacoreBuild.Tests.csproj```. 

Pour accéder aux nouvelles tâches, nous devons ajouter la directive ```require 'albacore'``` au début de notre fichier Rakefile. Ajoutons-le au même niveau que notre fichier solution.

``` ruby
require 'albacore'
```

Pour notre première action, nous allons chercher à construire notre projet. Pour cela, Albacore défini une tâche build qui va agir en utilisant MSBuild ou Mono en fonction de votre environnement. Il nous suffit ensuite d'indiquer les paramètres de compilation comme la solution, les cibles et la configuration : 

``` ruby
build :build do |b|
  b.file   = 'AlbacoreBuild.sln' 
  b.target = ['Clean', 'Rebuild']              
  b.prop 'Configuration', 'Release'            
  b.logging = 'minimal'                       
  b.nologo          
end
```

Pour tester, rien de plus simple, lançons juste la tâche avec ```rake build``` :

![Image](/images/posts/albacore/Capture-3.PNG)

Passons maintenant à la restauration des packages nuget et au lancement des tests. Le principe est le même que pour la tâche build : nous allons directement réutiliser les tâches existantes ```nugets_restore``` et ```test_runner```.

``` ruby
nugets_restore :restore do |p|
  p.out       = 'packages'      
  p.exe       = '.nuget/NuGet.exe' 
end

test_runner :tests do |tests|
  tests.files = FileList['AlbacoreBuild.Tests/bin/Release/AlbacoreBuild.Tests.dll'] 
  tests.exe = 'packages/NUnit.Runners.2.6.4/tools/nunit-console.exe' 
  tests.native_exe 
end
```

La configuration est plutôt simple : nous précisons les localisations de NuGet et du dossier devant contenir les packages. Puis ceux de la console NUnit et de l'assembly avec nos tests.

Comme nous l'avons vue précédemment, il est possible de lancer de manière unitaire une tâche avec la commande ```rake task_name```.  Il est aussi possible de spécifier une tâche par défaut dans le cas d'un appel à rake sans paramètre. Dans notre exemple, nous souhaitons restaurer, construire et tester notre solution.

``` ruby
task :default => [:restore, :build, :tests]
```

Au final, notre Rakefile complet ressemble à ceci :

``` ruby
require 'albacore'

nugets_restore :restore do |p|
  p.out       = 'packages'      
  p.exe       = '.nuget/NuGet.exe' 
end

build :build do |b|
  b.file   = 'AlbacoreBuild.sln' 
  b.target = ['Clean', 'Rebuild']              
  b.prop 'Configuration', 'Release'            
  b.logging = 'minimal'                       
  b.nologo          
end

test_runner :tests do |tests|
  tests.files = FileList['AlbacoreBuild.Tests/bin/Release/AlbacoreBuild.Tests.dll'] 
  tests.exe = 'packages/NUnit.Runners.2.6.4/tools/nunit-console.exe' 
  tests.native_exe 
end

task :default => [:restore, :build, :tests]
```

Et voilà, nous sommes maintenant capables de contrôler notre chaîne de build à travers un DSL simple et interopérable.

![Image](/images/posts/albacore/Capture-2.PNG)

N'hésitez pas à faire un tour sur leur [wiki](https://github.com/Albacore/albacore/wiki) pour plus d'information :)