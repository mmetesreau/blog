+++
author = "Mickael Metesreau"
title = "Heroku, AngularJs, Nancyfx... (Partie 1)"
date = "2014-10-02"
description = "Heroku, AngularJs, Nancyfx... (Partie 1)"
tags = []
+++

Dans cette série, je vous propose la création d'une application de [tableaux Kanban](http://www.infoq.com/articles/agile-kanban-boards) en sortant un peu des sentiers battus avec [AngularJs](https://angularjs.org/), [NancyFx](http://nancyfx.org/), [Petapoco](http://www.toptensoftware.com/petapoco/), [Heroku](https://www.heroku.com/)...

Cette première partie sera dédiée au setup de notre solution NancyFx sur Heroku.

### Prérequis 

Pour la suite de cet article, vous devez :

- ouvrir un compte sur [Heroku](http://heroku.com/) 
- installer [Heroku Toolbelt](https://toolbelt.heroku.com/)
- installer Visual Studio ou [MonoDevelop](http://monodevelop.com/)


### NancyFx

[NancyFx](http://nancyfx.org/) est un framework web alternatif pour .NET et Mono. Il permet de construire des services HTTP facilement, rapidement à la manière d'[Express](http://expressjs.com/) ou de [Sinatra](http://www.sinatrarb.com/).

IIS ne tournant pas sur Linux :p, nous allons utiliser un des composants du projet Katana pour hoster notre application. Pour information, Katana est une des implémentations de la spécification [OWIN](http://owin.org/) et je vous conseille d'y jeter un coup d’œil si vous l'avez loupé.

Créons une nouvelle application de type console :

![Image](/images/posts/heroku-angularjs-nancyfx-p1/image1.PNG)

Ajoutons les packages suivant avec NuGet :

- Install-Package Microsoft.Owin.Hosting -Version 2.1.0
- Install-Package Microsoft.Owin.Host.HttpListener -Version 2.1.0
- Install-Package Nancy.Owin 

Ajoutons le fichier Startup.cs :

``` csharp
class Startup
{
  public void Configuration(IAppBuilder app)
  {
    app.UseNancy();
  }
}
```

Modifions le fichier Program.cs :

``` csharp
class Program
{
  private static ManualResetEvent _quitEvent = new ManualResetEvent(false);

  static void Main(string[] args)
  {
    int port = 5000;

    if (args.Length > 0)
    {
      int.TryParse(args[0], out port);
    }
    Console.CancelKeyPress += (sender, eArgs) =>
    {
      _quitEvent.Set();
      eArgs.Cancel = true;
    };

    using (WebApp.Start<Startup>(string.Format("http://*:{0}", port)))
    {
      Console.WriteLine("Started");
      _quitEvent.WaitOne();
    }
  }
}
```

Ajoutons maintenant un module Nancy HomeModule.cs :

``` csharp
public class HomeModule : NancyModule
{
  public HomeModule()
  {
    Get["/status"] = _ => "I am alive";
  }
}
```

Lancez l'application en tant qu'administrateur et allez sur http://127.0.0.1:5000/status avec votre navigateur préféré.

![Image](/images/posts/heroku-angularjs-nancyfx-p1/image2.PNG)

### Heroku

Ajoutons un fichier Procfile à la racine de notre solution. Ce fichier sert à expliquer à Heroku comment démarrer notre application et le port sur lequel notre application doit écouter :

``` bash
web: mono KanbanBoard.exe $PORT
```

De la même façon, ajoutons un fichier .gitignore pour ne pas commit les dossiers packages, bin et obj :

``` text
[Oo]bj/
[Bb]in
[Dd]ebug*/
[Rr]elease*/
packages/
*.nupkg
```

Activons le restore package de NuGet :

![Image](/images/posts/heroku-angularjs-nancyfx-p1/image3.PNG)

Et maintenant, déployons le tout. Ouvrons une invite de commande :

Initialisation du dépôt Git :

``` bash
$ git init
```

Connexion à Heroku :

``` bash
$ heroku login
```

Création de l'application sur Heroku et configuration du dépôt distant :

``` bash
$ heroku apps:create mickael-metesreau-kanban-board
$ git remote -v
```

![Image](/images/posts/heroku-angularjs-nancyfx-p1/image4.PNG)

Les applications utilisant Mono n'étant pas prises en charge nativement par Heroku, il est nécessaire de spécifier comment construire notre application :

``` bash
$ heroku config:add BUILDPACK_URL=https://github.com/friism/heroku-buildpack-mono/
```

Il ne nous reste plus qu'à commit puis push sur Heroku :

Commit des fichiers de notre solution :

``` bash
$ git add -A
$ git commit -m "Initial commit"
```

Déploiement :

``` bash
$ git push heroku master
```

![Image](/images/posts/heroku-angularjs-nancyfx-p1/image5.PNG)

Une fois la résolution des dépendances et le build fait, l'application est [disponible](http://mickael-metesreau-kanban-board.herokuapp.com/status) sur heroku :)

![Image](/images/posts/heroku-angularjs-nancyfx-p1/image6.PNG)

### Conclusion

Déployer une application  Mono sur Heroku est extrêmement simple et rapide. Je trouve la centralisation de la gestion du versionning et du déploiement avec Git malin et très plaisant à utiliser. Enfin, Heroku offre pour moi l'avantage de disposer d'une infrastructure sans aucun coût afin de setup une application lors de mes développements.

Dans la seconde partie, nous allons formaliser l'API REST de notre système puis ajouter une couche de persistance avec Petapoco et Heroku Postgres.