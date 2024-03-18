+++
author = "Mickael Metesreau"
title = "Sublime Text & C#"
date = "2014-10-19"
description = "Sublime Text & C#"
tags = []
+++

Visual Studio est un IDE génial mais il n'est pas disponible sur toutes les plateformes et il peut parfois être long au démarrage. Dans cet article, je vous propose de voir comment utiliser Sublime Text pour coder en C# avec [OmniSharpSublime](https://github.com/moonrabbit/OmniSharpSublime).


OmniSharpSublime est une extension utilisant [OmniSharpServer](https://github.com/nosami/OmniSharpServer). Elle apporte pas mal de fonctionnalités utiles : 

- auto-complétion
- aller à l'implementation
- formatter le document
- ajouter/supprimer des fichiers au csproj
- et bien plus ...

OmniSharpServer est simplement un wrapper HTTP autour d'une librairie d'analyse de code utilisée par SharpDevelop et MonoDevelop : [NRefactory](https://github.com/icsharpcode/NRefactory). Elle offre des APIs permettant d’accéder à l'arbre syntaxique, aux types system et aux informations sémantiques un peu à la manière de [Roslyn](http://msdn.microsoft.com/en-us/vstudio/roslyn.aspx).

Si vous souhaitez en savoir plus à ce sujet, je vous conseille le lien [suivant](http://www.codeproject.com/Articles/408663/Using-NRefactory-for-analyzing-Csharp-code).

### Prérequis 

Pour la suite de cet article, vous devez avoir Python, Git et Sublime Text 3 sur votre poste.

### Installation

Edit : Il est maintenant possible d'installer le [package](https://sublime.wbond.net/packages/OmniSharp) directement depuis Sublime.

Clonons le dépôt Git dans le dossier Packages de Sublime Text :

``` shell
cd "C:\Users\{username}\AppData\Roaming\Sublime Text 3\Packages"
git clone https://github.com/moonrabbit/OmniSharpSublime.git
cd OmniSharpSublime
git submodule update --init --recursive
```

Compilons maintenant OmniSharpServer :

``` shell
cd OmniSharpServer
C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild .\OmniSharp.sln
```

### Création du projet

Ouvrons Sublime Text et créons un nouveau projet : *Project -> Save Project As*

Puis éditons le : *Project -> Edit Project*

Ce fichier va nous permettre de configurer Omnisharp :

``` json
{
	"folders":
	[{
		"name": "CSharpIsSublime",
		"follow_symlinks": true,
		"path": ".",
		"file_exclude_patterns":
		[
			"*.meta",
			"*.png",
			"*.dll",
			"*.mdb"
		],
		"folder_exclude_patterns":
		[
			"Library"
		]
	}],
	"settings":
	{
		"tab_size": 4
	},
	"solution_file": "./CSharpIsSublime.sln"
	}
```

Relançons Sublime Text afin de charger l'extension. Normalement vous devriez voir la chaîne **$omnisharp plugin_loaded** dans la console : *View -> Show console*

Ouvrons un fichier source, l’auto-complétion est maintenant disponible :)

![Image](/images/posts/csharp-et-sublime-text/image2.png)