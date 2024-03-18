+++
author = "Mickael Metesreau"
title = "Heroku, AngularJs, Nancyfx... (Partie 2)"
date = "2015-06-02"
description = "Heroku, AngularJs, Nancyfx... (Partie 2)"
tags = []
+++

Dans cette série, je vous propose la création d'une application de [tableaux Kanban](http://www.infoq.com/articles/agile-kanban-boards) en sortant un peu des sentiers battus avec [AngularJs](https://angularjs.org/), [NancyFx](http://nancyfx.org/), [Petapoco](http://www.toptensoftware.com/petapoco/), [Heroku](https://www.heroku.com/)

Dans la première [partie](/posts/heroku-angularjs-nancyfx-p1/), nous avons setup une solution NancyFx et l’avons déployée sur Heroku.

Cette seconde partie sera dédiée à la mise en place d'une API REST et d'une couche de persistance avec Petapoco et Heroku Postgres.

## Heroku Postgres

Heroku propose une version limitée et gratuite de [Postgres](https://addons.heroku.com/heroku-postgresql)

Nous allons provisionner cette extension pour notre application (mickael-metesreau-kanban-board pour ma part) en ligne de commande :

``` bash
$ heroku login

$ heroku addons:add heroku-postgresql:hobby-dev --app mickael-metesreau-kanban-board

$ heroku pg:credentials DATABASE --app mickael-metesreau-kanban-board
```

Notre base de données étant prête, nous scriptons la création de nos tables dans un fichier init.sql :

``` sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  password text,
  username text
);

CREATE TABLE tasks (
  id SERIAL PRIMARY KEY,
  userid integer REFERENCES users (id),
  title  text,
  description  text,
  creation date,
  status integer
);
```

Afin de se connecter à notre base, nous devons au préalable installer psql et l'ajouter à notre path. Il ne reste plus qu'à jouer les instructions sur notre instance :

``` bash
$ cat init.sql | heroku pg:psql --app mickael-metesreau-kanban-board 
```

## Notre application NancyFx

Ajoutons les packages suivants avec NuGet :

- Install-Package Nancy.Authentication.Token
- Install-Package PetaPoco
- Install-Package Npgsql


### Création des repositories

Ajoutons nos entités Task et User à notre solution :

``` csharp
public class Task
{
  public enum TaskStatus
  {
      TODO,
      DOING,
      DONE,
  }

  [Column("id")]
  public long Id { get; set; }
  [Column("userid")]
  public long Userid { get; set; }
  [Column("title")]
  public string Title { get; set; }
  [Column("description")]
  public string Description { get; set; }
  [Column("creation")]
  public DateTime Creation { get; set; }
  [Column("status")]
  public TaskStatus Status { get; set; }
}

public class User : IUserIdentity
{
  public long Id { get; set; }
  public string UserName { get; set; }
  public string Password { get; set; }

  [PetaPoco.Ignore]
  public IEnumerable<string> Claims { get; set; }
}
```

Afin de nous intégrer dans le modèle d'authentification de NancyFx, notre classe User doit implémenter l'interface IUserIdentity.

Créons maintenant les repositories en utilisant :

``` csharp
public class Repository
{
  protected Database db;
  private const string BDD_NAME = "Postgres";

  public Repository()
  {
    db = new PetaPoco.Database(BDD_NAME);
  }
}

public class Board : Repository
{
  private const string TABLE_NAME = "tasks";
  private const string PK_COLUMN = "id";
  private const string QUERY_SELECT_BY_USERID = "SELECT * FROM tasks where userid=@userId";

  public void Insert(Task task)
  {
    db.Insert(TABLE_NAME, PK_COLUMN, true, task);
  }

  public void Update(Task task)
  {
    db.Update(TABLE_NAME, PK_COLUMN, task);
  }

  public object GetAllByUserId(long userId)
  {
    return db.Query<Task>(QUERY_SELECT_BY_USERID,new {userId = userId}).ToList();
  }
}

public class Users : Repository
{
  private const string QUERY_SELECT_BY_USERNAME_AND_PASSWORD = "SELECT * FROM users where username=@username and password=@password";

  public User GetOneByUserNameAndPassword(string username, string password)
  {
    return db.FirstOrDefault<User>(QUERY_SELECT_BY_USERNAME_AND_PASSWORD, new { username = username, password=password });
  }
}
```

Petapoco se veut simple, rapide et léger. C'est un micro ORM qui s'occupe principalement de faire le mapping objet relationnel. La couche d’abstraction est très petite et on retrouve la "joie" de refaire un peu de SQL :)

Pour finir, ajoutons notre chaîne de connexion et notre provider Postgres dans le fichier app.config :

``` bash
$ heroku config --app mickael-metesreau-kanban-board | grep HEROKU_POSTGRESQL
```

``` xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.data>
    <DbProviderFactories>
      <add name="Npgsql Data Provider" invariant="Npgsql" support="FF" description=".Net Framework Data Provider for Postgresql Server" type="Npgsql.NpgsqlFactory, Npgsql, Version=2.1.3.0, Culture=neutral, PublicKeyToken=5d8b90d52f46fda7" />
    </DbProviderFactories>
  </system.data>
  <connectionStrings>
    <add name="Postgres" connectionString="Server=******;Port=****;User Id=******;Password=******;Database=******;SSL=true;" providerName="Npgsql"/>
  </connectionStrings>
</configuration>
```

### Ajout de l'authentification

NancyFx propose plusieurs [mécanismes d'authentification](https://github.com/NancyFx/Nancy/wiki/Authentication-overview). Dans notre cas, nous allons utiliser l'authentification par token. 

Ajoutons le fichier Bootstrapper pour ajouter cette fonctionnalité dans le pipeline de traitement des requêtes :

``` csharp
public class Bootstrapper : DefaultNancyBootstrapper
{
  protected override void RequestStartup(TinyIoCContainer container, IPipelines pipelines, NancyContext context)
  {
    TokenAuthentication.Enable(pipelines, new TokenAuthenticationConfiguration(container.Resolve<ITokenizer>()));
  }
}
```

### Mise en place de l'api


- /api/tasks        | GET |    Permet de récupérer toutes les tâches
- /api/tasks        | POST |  Permet d'ajouter une nouvelle tâche
- /api/tasks/{id}   | PUT |    Permet de modifier une tâche 
- /api/auth          | POST |  Permet de générer un token d'authentification

Ajoutons maintenant le fichier TasksApiModule.cs. Pour rappel, avec Nancyfx, les modules sont un peu l'équivalent des contrôleurs ASP.NET MVC. Ils vont nous permettre de router notre trafic HTTP vers nos méthodes.

``` csharp
public class ApiModule : NancyModule
{
  private Board board;
  private Users users;
  private ITokenizer tokenizer;

  public ApiModule(Board board, Users users, ITokenizer tokenizer)
      : base("/api")
  {
    this.board = board;
    this.users = users;
    this.tokenizer = tokenizer;

    Get["/tasks"] = GetTasks;
    Post["/tasks"] = PostTask;
    Put["/tasks/{id:int}"] = PutTask;
    Post["/auth"] = PostAuth;
  }

  private Response PostAuth(dynamic p)
  {
      var username = (string)this.Request.Form.Username;
      var password = (string)this.Request.Form.Password;

      var userIdentity = users.GetOneByUserNameAndPassword(username,password);

      if (userIdentity == null)
      {
          return HttpStatusCode.Unauthorized;
      }

      userIdentity.Claims = new List<string>() { userIdentity.Id.ToString() };

      var token = tokenizer.Tokenize(userIdentity, Context);

      return Response.AsJson(new
      {
          Token = token
      });
  }

  private long GetCurrentUserIdFromClaims()
  {
      var user = this.Context.CurrentUser;

      var id = long.Parse(user.Claims.First());

      return id;
  }

  private Response PostTask(dynamic p)
  {
      this.RequiresAuthentication();

      var task = this.Bind<Task>();

      task.Creation = DateTime.Now;
      task.Userid = GetCurrentUserIdFromClaims();

      board.Insert(task);

      return Response.AsJson(task, HttpStatusCode.OK);
  }

  private Response PutTask(dynamic p)
  {
      this.RequiresAuthentication();

      var task = this.Bind<Task>();

      task.Userid = GetCurrentUserIdFromClaims();

      board.Update(task);

      return Response.AsJson(task, HttpStatusCode.OK);
  }

  private Response GetTasks(dynamic p)
  {
      this.RequiresAuthentication();

      var userId = GetCurrentUserIdFromClaims();

      var tasks = board.GetAllByUserId(userId);

      return Response.AsJson(tasks, HttpStatusCode.OK);
  }
}
```

Vous avez peut être remarqué que l'instanciation des dépendances de notre module est déléguée. Par défaut, NancyFx utilise le conteneur IOC [TinyIOC](https://github.com/grumpydev/TinyIoC) pour cela.

La méthode PostAuth récupère les valeurs username et password du formulaire et retourne une 401 si l'utilisateur n'existe pas. Dans le cas contraire, on génère un token pour le client. 

Dans notre exemple, les claims sont initialisées juste avant de générer le token. Cela nous permettra d'obtenir l'id de l'utilisateur dans notre contexte à chaque appel fait avec ce jeton.

Les méthodes GetTasks, PutTask et PostTask sont plutôt simples. On se contente de brancher notre repository en ayant appelé au préalable RequiresAuthentication.

### Déploiement & Tests

Pour déployer, il nous suffit de commit puis de push sur Heroku :

``` bash
$ git add -A
$ git commit -m "Step 2 - Api & Postgres"
$ git push heroku master
```

Ajoutons un utilisateur de test :

``` bash
$ heroku pg:psql --app mickael-metesreau-kanban-board

=> insert into users ("password","username") values ('Passw0rd','mickael');
```

Et maintenant un peu de [curl](http://curl.haxx.se/) pour tester tout ça :

``` bash
$ curl http://mickael-metesreau-kanban-board.herokuapp.com/api/tasks

curl : Le serveur distant a retourné une erreur : (401) Non autorisé.

$ curl --data "Username=mickael&Password=Passw0rd" http://mickael-metesreau-kanban-board.herokuapp.com/api/auth

{"token":"dG90bw0KDQo2MzU2NTY0OTIyNDQ1Nzk2MDkNCk1vemlsbGEvNS4wIChXaW5kb3dzOyBVOyBNU0lFIDkuMDsgV0luZG93cyBOVCA5LjA7IGVuLVVTKSk=:Va4T6sVzXGRfA0FYBseh17EFn8mO719I2nG+SxxSOn8="}   

$ curl --header "Authorization: Token dG90bw0KDQo2MzU2NTY0OTIyNDQ1Nzk2MDkNCk1vemlsbGEvNS4wIChXaW5kb3dzOyBVOyBNU0lFIDkuMDsgV0luZG93cyBOVCA5LjA7IGVuLVVTKSk=:Va4T6sVzXGRfA0FYBseh17EFn8mO719I2nG+SxxSOn8="  http://mickael-metesreau-kanban-board.herokuapp.com/api/tasks

[]

$ curl --data "Status=0&Description=desc&Title=title" --header "Authorization: Token dG90bw0KDQo2MzU2NTY0OTIyNDQ1Nzk2MDkNCk1vemlsbGEvNS4wIChXaW5kb3dzOyBVOyBNU0lFIDkuMDsgV0luZG93cyBOVCA5LjA7IGVuLVVTKSk=:Va4T6sVzXGRfA0FYBseh17EFn8mO719I2nG+SxxSOn8="  http://mickael-metesreau-kanban-board.herokuapp.com/api/tasks

$ curl --header "Authorization: Token dG90bw0KDQo2MzU2NTY0OTIyNDQ1Nzk2MDkNCk1vemlsbGEvNS4wIChXaW5kb3dzOyBVOyBNU0lFIDkuMDsgV0luZG93cyBOVCA5LjA7IGVuLVVTKSk=:Va4T6sVzXGRfA0FYBseh17EFn8mO719I2nG+SxxSOn8="  http://mickael-metesreau-kanban-board.herokuapp.com/api/tasks

[{"id":1,"userId":1,"title":"title","description":"desc","creation":"2015-03-03T00:00:00.0000000+01:00","status":0}]
```


## Conclusion

Formaliser une API avec NancyFx est assez simple. La mise en place de l'authentification n'est que l'ajout d'une middleware dans le pipeline des requêtes. 

Quant à notre couche d'accès aux données, on revient à la base avec SQL en s'évitant la fastidieuse couche de mapping avec Petapoco. Pas de couche d’abstraction de haut niveau, on maîtrise de nouveau ce que l'on fait :)

Dans une troisième partie, nous verrons comment créer des tests pour notre API (oui ce n'est pas bien de ne pas avoir commencé par ça) puis nous créerons un client avec AngularJs.

Les sources de l'exemple sont disponibles [ici](https://github.com/mmetesreau/KanbanBoard).