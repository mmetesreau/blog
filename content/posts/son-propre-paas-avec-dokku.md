+++
author = "Mickael Metesreau"
title = "Son propre PaaS avec Dokku"
date = "2014-07-25"
description = "Son propre PaaS avec Dokku"
tags = []
+++

[Dokku](https://github.com/progrium/dokku) est un outil permettant de mettre en place un PaaS à la Heroku sur vos serveurs. Le but est de faciliter les livraisons en rendant le déploiement aussi simple que l'exécution d'une unique commande git :

> $ git push production master

Dokku unifie le processus de déploiement dans le cas d'un parc applicatif hétérogène. Vous souhaitez déployer une application PHP ? git push php-app master. Vous souhaitez déployer une application Java ? git push java-app master. Vous souhaitez déployer une application Node ? git push node-app master...

Dokku se compose de plusieurs briques :

- **Gitreceive :** interface git & ssh
- **Buildpack :** préparation des applications
- **Docker :** moteur de containers
- **Nginx :** gestion du trafic http

Voyons comment mettre en place Dokku et l'utiliser pour déployer une application [Nancyfx](https://github.com/NancyFx/Nancy).

### Prérequis 

Pour cet article, j'utilise :

- Une machine virtuelle Ubuntu 14.04 LTS avec openssh-server installé
- Un terminal disposant de git et de ssh 

La machine virtuelle dispose d'internet et est accessible depuis mon ordinateur au travers du nom de domaine mydokku.net. L'application sera disponible sur le sous domaine helloworld.mydokku.net.

L'ensemble des entrées DNS est configuré dans mon fichier HOSTS :

``` bash
192.168.101.130 helloworld.mydokku.net
192.168.101.130 mydokku.net
```

Pour la suite, n'oubliez pas d'adapter les commandes en fonction de vos entrées DNS.

### Installation et configuration de Dokku 

Générons depuis notre poste local une clef ssh. Elle va nous permettre de nous connecter à notre serveur sans mot de passe et plus tard de pousser nos applications sur Dokku.

``` bash
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
$ Enter file in which to save the key (/home/[Utilisateur]/.ssh/id_rsa):
$ Enter passphrase (empty for no passphrase):
$ Enter same passphrase again:
Your identification has been saved in id_rsa.
Your public key has been saved in id_rsa.pub.
```

Ajoutons la clef publique sur notre serveur :

``` bash
$ cat ~/.ssh/id_rsa.pub | ssh [utilisateur]@[mydokku.net] 'cat >> ~/.ssh/authorized_keys'
```

Connectons nous en ssh sur votre vm (sans mot de passe ^^) et lançons la commande suivante afin d'installer Dokku :

``` bash
$ ssh [utilisateur]@[mydokku.net]
$ wget -qO- https://raw.github.com/progrium/dokku/v0.2.3/bootstrap.sh | sudo DOKKU_TAG=v0.2.3 bash
```

Une fois l'installation terminée, configurons Dokku pour utiliser notre clef publique. La documentation propose de lancer cette commande :

``` bash
$ cat ~/.ssh/id_rsa.pub | ssh [utilisateur]@[mydokku.net] 'sudo sshcommand acl-add dokku progrium'
```

Utilisant Cygwin, j'ai dû, pour ma part, uploader la clef publique puis lancer directement la commande sur le serveur :

``` bash
$ cat ~/.ssh/id_rsa.pub | ssh [utilisateur]@[mydokku.net] 'cat >> ~/id_rsa.pub'
$ ssh [utilisateur]@[mydokku.net]
$ cat ~/id_rsa.pub | sudo sshcommand acl-add dokku progrium
```

Dokku cherche à déployer les applications dans des sous domaine du serveur. Pour cela, nous devons renseigner le domaine dans le fichier /home/[utilisateur]/dokku/VHOST :

``` bash
$ ssh [utilisateur]@[mydokku.net]
$ cd /home/dokku
$ sudo su
# echo "[mydokku.net]" > VHOST
# exit
```

Enfin, comme nous souhaitons déployer des applications utilisant Mono, nous devons installer un plugin spécifique :

``` bash
$ ssh [utilisateur]@[mydokku.net]
$ cd /var/lib/dokku/plugins
$ sudo git clone https://github.com/pauldub/dokku-multi-buildpack
$ sudo dokku plugins-install
```

### L'application

NancyFx est un micro-framework web open source pour .NET et Mono. Il permet de construire des applications web en utilisant un DSL proche des verbes HTTP.

``` csharp
public class SampleModule : Nancy.NancyModule
{
  public SampleModule()
  {
    Get["/"] = _ => "Hello World!";
  }
}
```

Le but n'étant pas de faire un tutoriel exhaustif sur NancyFx, nous allons utiliser l'application disponible [ici](https://github.com/mmetesreau/nancyfx-dokku). Cette application se contente de renvoyer la chaîne HelloWorld lorsqu'on cherche à atteindre la route /.

Pour plus d'informations sur ce framework, je vous renvoie à la présentation [suivante](http://www.d80.co.uk/post/2013/01/04/NancyFx-Tutorial.aspx).

``` bash
$ wget https://github.com/mmetesreau/nancyfx-dokku/archive/master.zip && unzip master.zip && rm master.zip
```

Les applications utilisant Mono n'étant pas prises en charge nativement par Dokku, il a été nécessaire d'ajouter deux fichiers particuliers à la racine du projet.

Le fichier .buildpacks spécifiant comment votre application va être construite et déployée :

```
https://github.com/wilfrem/heroku-buildpack-mono
```

Le fichier Procfile permettant de déclarer les commandes démarrant l'application :

```
web: mono HelloWorld.exe $PORT
```

### Déploiement

Afin de déployer l'application, préparons notre dépôt git :

``` bash
$ git init
$ git add .buildpacks 
$ git commit -m "Configure buildpacks"
$ git add Procfile
$ git add HelloWorld.sln
$ git add HelloWorld
$ git add .nuget
$ git commit -m "Add sources"
```

Déployons maintenant notre application sur notre serveur Dokku :

``` bash
$ git push dokku@[mydokku.net]:helloworld master
```

Si tout se passe correctement, l'application devrait être accessible à l'adresse suivante : helloworld.[mydokku.net].

L'uri est déduite à partir du fichier VHOST (domaine) et du nom du dépôt dokku (sous domaine). 

``` bash
Time Elapsed 00:00:59.0930260
-----> Build SUCCESS
Using release configuration from last framework .NET:
addons: []
default_process_types: {}
-----> Discovering process types
Procfile declares types -> web
-----> Releasing mononancy ...
-----> Deploying mononancy ...
=====> Application deployed:
http://helloworld.mydokku.net
```

Attention, dans le cas de notre application NancyFX, il faut faire attention à spécifier la bonne uri dans le fichier app.config pour éviter de s'exposer à l'erreur **bad request (invalid hostname)**. 

Dans mon cas, cela donne :

``` xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <appSettings>
      <add key="host" value="http://helloworld.mydokku.net" />
    </appSettings>
</configuration>
```

A chaque push, Dokku télécharge les dépendances, compile notre application et déploye dans un conteneur docker :

``` bash
$ ssh [utilisateur]@[mydokku.net]
$ sudo docker ps
IMAGE COMMAND PORTS
dokku/helloworld:latest /bin/bash -c '/start 0.0.0.0:49172->5000/tcp
```

La configuration du virtual host Nginx est disponible dans le dossier : /home/dokku/[application].

``` bash
$ ssh [utilisateur]@[mydokku.net]
$ cat /home/dokku/helloworld/nginx.conf
```

Il est facile d'afficher la sortie standard de l'application pour debugger avec la commande : dokku logs [application].

``` bash
$ ssh [utilisateur]@[mydokku.net]
$ dokku logs helloworld 
Listening at helloworld.mydokku.net:5000
```

Vous pouvez aussi gérer vos applications directement grâce aux commandes suivantes :

- dokku delete [application]
- dokku deploy [application] 

Pour finir, si vous souhaitez utiliser directement Katana à la place de Nancyfx, un exemple est disponible [ici](https://github.com/wilfrem/KatanaTest).