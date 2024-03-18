+++
author = "Mickael Metesreau"
title = "Jouer avec F# et Sublime Text 3"
date = "2015-07-29"
description = "Jouer avec F# et Sublime Text 3"
tags = []
+++

J'adore Sublime Text car il a la particularité de se lancer vite, très vite. Et comme le sujet en vogue depuis quelque temps est la programmation fonctionnelle, je cherchai un moyen de m'amuser avec F# sans passer par Visual Studio. 

### Prérequis 

Si vous ne l'avez pas déjà fait, la première étape pour profiter pleinement de Sublime Text est d'installer le package manager. Dans la console de ST3 ```View > Show Console```, collez les lignes suivantes :

``` shell
import urllib.request,os,hashlib; h = 'eb2297e1a458f27d836c04bb0cbaf282' + 'd0e7a3098092775ccb37ca9d6b2e4b7d'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```

Si vous utilisez encore ST2, le détail des instructions se trouve [ici](https://packagecontrol.io/installation).

Puisque vous lisez ce poste, j'en déduis que vous avez F# installé sur votre machine (Si ce n'est pas le cas alors vous loupez un truc :p). Vérifiez que les outils du SDK sont disponibles depuis votre path et si ce n'est pas le cas, ajoutez-les. Chez moi en PowerShell cela donne :

``` shell
[Environment]::SetEnvironmentVariable("Path",$env:Path + ";C:\Program Files (x86)\Microsoft SDKs\F#\3.1\Framework\v4.0","Machine")
```

Maintenant que l'environnement est configuré, il est temps de faire un petit tour des extensions disponibles.

### Sublime FSharp Package

La première extension est [sublime-fsharp-package](https://github.com/fsharp/sublime-fsharp-package). Elle a l'avantage d'être autosuffisante et la plus récente de toutes. Un coup de ```Ctrl+Shift+P``` pour appeler la palette de commande, puis ```Install Package > sublime fsharp package```. 

Une fois l'installation terminée, vous allez trouver les fonctionnalités bien pratiques suivantes :

- ```Ctrl+k, Ctrl+i	``` pour les tooltips
- ```Ctrl+space	``` pour l'auto-complétion
- ```F12``` Aller à la définition
- ```F7``` Exécuter le script
- Surlignage syntaxique et Live error checking

Pour finir, il est possible d'utiliser la palette de commande avec ```Ctrl+., Ctrl+.``` pour afficher toutes les commandes F#.

![Image](/images/posts/jouer-avec-fsharp-et-sublime-text/st3-fsharp1.png)

Cependant, il manque pour moi la possibilité de jouer facilement avec un [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) comme on peut le faire dans Visual Studio et ainsi tester du code en temps réel.

### Sublimetext FSharp & SublimeREPL

Comme vous l'avez facilement deviné, il est possible de résoudre le problème du REPL avec l'ajout de certaines extensions :

- [sublimetext-fsharp](https://github.com/hoest/sublimetext-fsharp)
- [SublimeREPL](https://github.com/wuub/SublimeREPL)

Celle-ci vont vous permettre d'afficher une seconde vue qui contiendra un wrapper sur F# interactive et de le nourrir avec des raccourcies clavier :

- ```Ctrl+,, s``` Evaluer la sélection
- ```Ctrl+,, f``` Evaluer le fichier
- ```Ctrl+,, l``` Evaluer la ligne
- ```Ctrl+,, b``` Evaluer le bloc

![Image](/images/posts/jouer-avec-fsharp-et-sublime-text/fsharp-st3-2.png)

Pour afficher le REPL, cliquez sur ```Tools > SublimeREPL > F#``` et/ou ajoutez un nouveau raccourci en éditant  ```Preferences | Key bindings – user``` :

``` json
[
    {
        "keys": [
            "ctrl+alt+f"
        ], 
        "args": {
            "id": "repl_f#", 
            "file": "config/F/Main.sublime-menu"
        }, 
       "command": "run_existing_window_command"
    }
]
```

Enfin, il peut être pratique de configurer Sublime Text pour utiliser la même combinaison que Visual Studio ```Alt+Enter``` toujours en éditant  ```Preferences | Key bindings – user``` :

``` json
{
    "keys": [
       "alt+enter"
    ], 
    "args": {
       "scope": "selection"
    }, 
    "command": "repl_transfer_current"
}
```

### Conclusion

Il est facile d'avoir un environnement rapide et simple pour coder en F# avec Sublime Text même si, bien entendu, il n'est pas (encore?) aussi intégré que Visual Studio. Maintenant il ne vous reste plus qu'à bien vous amuser et à (re) visionner la session de Jérémie Chassain [IF YOU'RE NOT LIVE CODING, YOU'RE DEAD CODING!](http://videos.ncrafts.io/page/3).