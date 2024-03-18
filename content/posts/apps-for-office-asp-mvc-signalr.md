+++
author = "Mickael Metesreau"
title = "Apps For Office, ASP MVC et SignalR"
date = "2015-02-23"
description = "Apps For Office, ASP MVC et SignalR"
tags = []
+++

Avec les sorties d'Office 2013 et d'Office Web App est apparu un nouveau moyen d'étendre les capacités de notre suite bureautique préférée : les App For Office.

Techniquement, une App for Office est simplement une page web accessible depuis une Iframe intégrée à Excel, Word, PowerPoint... Le tout interagissant au travers une api Javascript créée pour l'occasion.

Dans cet article nous allons ajouter une fonctionnalité de partage en temps réel à Excel 2013.

Il existe trois types d'Apps For Office : Task pane apps, Content apps, Mail apps :

- Excel 2013 : Task pane, Content
- Excel Web App : Task pane, Content
- Word 2013 : Task pane
- Outlook 2013 : Mail
- Outlook Web App : Mail
- PowerPoint 2013 : Task pane
- Project Professional 2013 : Task pane

Une description plus complète est disponible [ici](http://msdn.microsoft.com/en-us/library/jj220082.aspx).

### Prérequis 

Pour cet article, j'utilise :

- Visual Studio 
- ASP.NET MVC 
- Office 2013

### Hello World

La première chose à faire est de créer le site qui hébergera l'App for Office. Il est possible d'utiliser des templates disponibles dans Visual Studio. Cependant, dans notre cas, nous allons baser notre solution sur un projet ASP.NET MVC vide.

Lançons Visual Studio, créons un nouveau projet ASP.NET Mvc vide AppForExcel.ShareYourSheets

![Image](/images/posts/apps-for-office-asp-mvc-signalr/image1.png)

Ajoutons un contrôleur Home et une vue associée :

``` csharp
public class HomeController : Controller
{
  public ActionResult Index()
  {
    return View();
  }
}
```

``` html
<html>
  <head>
    <title>ShareYourSheets</title>
  </head>
  <body>
    <div>Hello World!!</div>
  </body>
</html>
```


Pour tester notre Hello World, il est nécessaire de créer un [manifest](https://msdn.microsoft.com/en-us/library/office/fp161044.aspx). Ce fichier permettra d'indiquer à Excel les paramètres permettant d'utiliser notre App (url, nom, type,...)

Ajoutons le fichier ShareYourSheets.xml puis activons le partage réseau sur ce fichier :

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!--Created:cb85b80c-f585-40ff-8bfc-12ff4d0e34a9-->
<OfficeApp xmlns="http://schemas.microsoft.com/office/appforoffice/1.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="TaskPaneApp">
  <Id>d8a1718e-417d-4845-9c90-26b1814a0a2b</Id>
  <Version>1.0.0.0</Version>
  <ProviderName>Mickael Metesreau</ProviderName>
  <DefaultLocale>en-US</DefaultLocale>
  <DisplayName DefaultValue="ShareYourSheets" />
  <Description DefaultValue="ShareYourSheets Description"/>
  <Capabilities>
    <Capability Name="Workbook" />
  </Capabilities>
  <DefaultSettings>
    <!--A remplacer par l'url de votre application ;) -->
    <SourceLocation DefaultValue="http://localhost:1337/" />
  </DefaultSettings>
  <Permissions>ReadWriteDocument</Permissions>
</OfficeApp>
```

Lançons notre projet puis ouvrons Excel 2013.

Dans File, Options, Trust Center, Trust Center Settings, Trusted App Catalogs, nous allons indiquer l'emplacement réseau de notre catalogue :

\\ \\ [Nom-Machine] \\ [Path-Manifest] \ et cocher l'option Show in Menu :

![Image](/images/posts/apps-for-office-asp-mvc-signalr/image2-4.png)

Pour que les changements soient pris en compte, nous devons redémarrer Excel.

Créons un nouveau classeur, puis depuis le ribbon cliquons sur Insert, Apps for Office, See All. Dans l'onglet Shared Folder, sélectionnons notre application :

![Image](/images/posts/apps-for-office-asp-mvc-signalr/image3-1.png)

Vous venez de créer votre première App for Office :)

![Image](/images/posts/apps-for-office-asp-mvc-signalr/image4-1.png)

### Un peu de temps réel

Nous allons maintenant préparer notre solution pour permettre à nos utilisateurs de communiquer en temps réel. Nous allons ici utiliser la bibliothèque [SignalR](http://signalr.net/) qui se chargera d’établir un canal de communication entre le serveur et le client.

Ajoutons dans la solution le package SignalR avec Nuget :

- Install-Package Microsoft.AspNet.SignalR

Ajoutons une classe Notifier. Cette classe hérite de la classe Hub de SignalR afin de gérer l'envoi de messages aux clients depuis le serveur :

``` csharp
public class Notifier : Hub
{
  private static List<string> users = new List<string>();

  public override Task OnDisconnected(bool stopCalled)
  {
    return base.OnDisconnected(stopCalled).ContinueWith((param) =>
    {
      if (users.Contains(Context.ConnectionId))
      {
        users.Remove(Context.ConnectionId);
        Clients.All.Users(users);
      }
    });
  }

  public override Task OnConnected()
  {
    return base.OnConnected().ContinueWith((param) =>
    {
      users.Add(Context.ConnectionId);
      Clients.All.Users(users);
    });
  }

  public void NotifyUser(string connectionId, string data)
  {
    Clients.Client(connectionId).ReceivedData(Context.ConnectionId, data, DateTime.Now.ToString("dd/MM/yyyy"));
  }
}
```

Les méthodes OnConnected et OnDisconnected, nous permettent d'envoyer un message aux clients, respectivement, à chaque nouvelle connexion et déconnexion. Nous pourrons ainsi tenir une liste des utilisateurs connectés coté client.

La méthode NotifyUser permet à un utilisateur d'envoyer un message à un autre utilisateur identifié par son connectionId.

SignalR reposant sur [Owin](http://owin.org/), ajoutons une classe Startup pour démarrer notre hub :

``` csharp
[assembly: OwinStartup(typeof(AppForExcel.ShareYourSheets.Startup))]
namespace AppForExcel.ShareYourSheets
{
  public class Startup
  {
    public void Configuration(IAppBuilder app)
    {
      app.MapSignalR();
    }
  }
}
```

Préparons maintenant notre vue index.cshtml :

- users-info : Contiendra la liste des utilisateurs actuellement connectés 
- connection-info : Contiendra l'id de l'utilisateur courant

``` html
<!DOCTYPE html>
<html>
  <head>
    <title>ShareYourSheets</title>

    <script type="text/javascript" src="scripts/jquery-1.10.2.min.js"></script>
    <script type="text/javascript" src="scripts/jquery.signalR-2.2.0.min.js"></script>
    <script type="text/javascript" src="signalr/hubs"></script>
    <script type="text/javascript">
    $(function () {
      var notifier = $.connection.notifier;
      notifier.client.users = function (lstUsers) {
        $("#users-info").empty();
        $.each(lstUsers, function (index, value) {
          if (value != $.connection.hub.id) {
            $("#users-info").append("<p>- user" + value + "</p>");
          }
        });
      };
      $.connection.hub.start().done(function () {
        $('#connection-info').empty();
        $('#connection-info').append('<strong>Connected as ' + $.connection.hub.id + '</strong>');
      });
    });
    </script>
  </head>
  <body>
    <h1>Share Your Sheets</h1>
    <div id="connection-info"></div>
    <div id="users-info">
    </div>
  </body>
</html>
```
En jouant avec plusieurs instances d'Excel, on constate que nous avons maintenant la liste des utilisateurs utilisant notre application en temps réel :) 

![Image](/images/posts/apps-for-office-asp-mvc-signalr/image5-1.png)

### Interactions entre Excel et notre application

A ce stade, nous arrivons à afficher une page web dans Excel et à communiquer en temps réel entre différentes instances de cette page.

Nous allons maintenant ajouter la possibilité à un utilisateur d'envoyer des données provenant directement de son classeur à un autre utilisateur.

Afin de réaliser cette fonctionnalité, nous avons besoin de communiquer entre notre app et Excel. Microsoft met à notre disposition la bibliothèque [office.js](https://appsforoffice.microsoft.com/lib/1.1/hosted/office.js).

Office.js expose simplement un ensemble d'api permettant d'interagir avec le contenu du document et dans notre cas nous allons utiliser les fonctions suivantes :

- Office.initialize permet de s'attacher à un événement lancé lorsque l'environnement est chargé et que l'application est prête à interagir avec le document
- Office.context.document donne accès un objet représentant le document courant
- Office.context.document.setSelectedDataAsync et Office.context.document.getSelectedDataAsync autorise respectivement l'écriture et la lecture de données sélectionnées par l'utilisateur dans le document

Créons une fonction getSelection qui va se charger de récupérer les cellules actuellement sélectionnées par l'utilisateur et une fonction paste qui va, quant à elle, insérer des cellules : 

``` js
context = Office.context.document;

function getSelection(success, error) {
  context.getSelectedDataAsync(Office.CoercionType.Matrix, function (asyncResult) {
    if (asyncResult.status != Office.AsyncResultStatus.Failed) {
      success(asyncResult.value);
    } else {
      error(asyncResult.error);
    }
  });
}

function paste(data, error) {
  context.setSelectedDataAsync(data, { coercionType: "matrix" }, function (asyncResult) {
    if (asyncResult.status == Office.AsyncResultStatus.Failed)) {
      error(asyncResult.error);
    }
  });
}
```

Modifions notre fichier index.cshtml pour intégrer nos nouvelles fonctions. Pour cela, nous allons rajouter un bouton pour partager avec un utilisateur et un clipboard pour interagir avec des données reçues :

``` js
<!DOCTYPE html>
<html>
<head>
  <title>ShareYourSheets</title>

  <script type="text/javascript" src="scripts/jquery-1.10.2.min.js"></script>
  <script type="text/javascript" src="scripts/jquery.signalR-2.2.0.min.js"></script>
  <script type="text/javascript" src="https://appsforoffice.microsoft.com/lib/1.1/hosted/office.js"></script>
  <script type="text/javascript" src="signalr/hubs"></script>
  <script type="text/javascript">
  Office.initialize = function () {
    $(document).ready(function () {

    context = Office.context.document;

    function paste(data, error) {
      context.setSelectedDataAsync(data, { coercionType: "matrix" }, function (asyncResult) {
        if (asyncResult.status == Office.AsyncResultStatus.Failed) {
          error(asyncResult.error);
        }
      });
    };

    function getSelection(success, error) {
      context.getSelectedDataAsync(Office.CoercionType.Matrix, function (asyncResult) {
        if (asyncResult.status != Office.AsyncResultStatus.Failed) {
          success(asyncResult.value);
        } else {
          error(asyncResult.error);
        }
      });
    }

    var notifier = $.connection.notifier;

    notifier.client.receivedData = function (user, data, date) {
      var elt = $("<div class='data'><p>At" + date + " from user " + user + "</p><p class='value'>" + data + "</p><p><input type='button' value='Add'></p></div>");

      elt.click(function (evt) {
        var data = JSON.parse($(this).find(".value").html());
        paste(data, function (error) {
          $("#error").val(error);
        });
      });
      $("#data-list").append(elt);
    };

    notifier.client.users = function (lstUsers) {
      $("#users-info").empty();
      $.each(lstUsers, function (index, value) {
        if (value != $.connection.hub.id) {
          $("#users-info").append("<p>- user" + value + " <input type='button' id='" + value + "' value='Send'></p>");

          $("#" + value + "").click(function (evt) {
            getSelection(function (data) {
              notifier.server.notifyUser(value, JSON.stringify(data));
            }, function (error) {
              $("#error").val(error);
            });
          });
        }
      });
    };
    $.connection.hub.start().done(function () {
      $('#connection-info').empty();
      $('#connection-info').append('<strong>Connected as ' + $.connection.hub.id + '</strong>');
    });
    });
  };
  </script>
</head>
<body>
  <h1>Share Your Sheets</h1>
  <div id="connection-info"></div>
  <div id="error"></div>
  <div id="users-info"></div>
  <div id="data-list"></div>
</body>
</html>
```

Il ne reste plus qu'à ouvrir plusieurs instances Excel et à jouer :

![screenshot2](/images/posts/apps-for-office-asp-mvc-signalr/screenshot2.gif)

Le projet complet est disponible [ici](https://github.com/mmetesreau/ShareYourSheets).