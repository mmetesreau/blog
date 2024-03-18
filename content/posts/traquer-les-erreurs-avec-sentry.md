+++
author = "Mickael Metesreau"
title = "Traquer les erreurs avec Sentry"
date = "2014-11-21"
description = "Traquer les erreurs avec Sentry"
tags = []
+++

Marre de lire des fichiers de logs ? [Sentry](https://getsentry.com) est une plateforme de logging d'événements et d'erreur open source. Simple, efficace et jolie Sentry s'utilise aussi bien en mode SAS que confortablement installé sur vos propres serveurs pour explorer, filtrer, prioriser et suivre ces étranges petites erreurs. 

Des SDKs existant dans pas mal de langages (JavaScript, Python, Ruby, ...), tentons de la consommer en C# avec de sympathiques projets Github :)

### Prérequis 

Dans cette article, j'utilise la version [trial](https://www.getsentry.com/pricing/) de l'offre SAS qui autorise 7 jours d'historique et 250 événements par jour.

### SharpRaven

[SharpRaven](https://github.com/getsentry/raven-csharp) est un client simple pour Sentry.

Une fois installé via Nuget :

- Install-Package SharpRaven -Version 1.4.2

il suffit d'instancier le client et d'utiliser les méthodes CaptureException et/ou CaptureMessage :

``` csharp
static void Main(string[] args)
{
  //La clef est dispo dans l'onglet application, clés d'api
  var ravenClient = new RavenClient("http://public:secret@example.com/project-id");

  ravenClient.CaptureMessage("Hello World!");
 
  try
  {
    throw new Exception("Shit happens");
  }
  catch (Exception e)
  {
    ravenClient.CaptureException(e);
  }
}
```

Et voilà le travail : 

![Image](/images/posts/traquer-les-erreurs-avec-sentry/image1--1-.PNG)

### SentryAppender

La seconde [librairie](https://github.com/themotleyfool/SentryAppender) est un appender pour [log4net](http://logging.apache.org/log4net/).

Installons la dépendance log4net :

- Install-Package log4net

En ce qui concerne SentryAppender, je n'ai pas l'impression que le package soit disponible depuis la galerie Nuget. J'ai donc du compiler le projet moi même.

Pour information, j'ai ouvert une issue [ici](https://github.com/themotleyfool/SentryAppender/issues/2).

Créons notre logger :

``` csharp
static void Main(string[] args)
{
  ILog logger = LogManager.GetLogger("Log4NetSentry");

  logger.Info("Hello World!");

  try
  {
    throw new Exception("Shit happens");
  }
  catch (Exception e)
  {
    logger.Fatal(e);
  }
  Console.ReadLine();
}
```

Il ne reste qu'à configurer le tout :

``` csharp
<configuration>
  <configSections>
    <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net"/>
  </configSections>
  <log4net>
    <root>
      <level value="DEBUG" />
      <appender-ref ref="RavenAppender" />
    </root>
      <appender name="RavenAppender" type="SharpRaven.Log4Net.SentryAppender, SharpRaven.Log4Net">
        <DSN value="http://public:secret@example.com/project-id" />
        <Logger value="Log4NetSentry" />
        <threshold value="ERROR" />
        <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%5level - %message%newline" />
      </layout>
    </appender>
  </log4net>
</configuration>

```

Peu de travail pour obtenir une si jolie timeline d'erreur :

![Image](/images/posts/traquer-les-erreurs-avec-sentry/image2--1-.PNG)

Et rappelez vous, logger, c'est bon pour la santé !