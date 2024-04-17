+++
author = "Mickael Metesreau"
title = "Vagrant & Azure"
date = "2014-11-19"
description = "Vagrant & Azure"
tags = []
+++

[Vagrant](https://www.vagrantup.com/) est un outil dont l'ambition est d'en finir avec la ô combien célèbre phrase : *"Mais ça marche sur ma machine"*. Il permet de construire de manière reproductible et portable des environnements de développement aussi simplement qu'en exécutant une ligne de commande :

> $ vagrant up

Historiquement, Vagrant est un wrapper au dessus de VirtualBox. Cependant il ne doit pas être vue comme un simple moyen de gérer des machines virtuelles. En partant du constat que nos postes de développement ne sont jamais ISO prod (OS différents, multiple versions de runtime, etc), Vagrant va nous permettre de créer des environnements isolés reproduisant la production et donc de tester notre code dans des conditions d'exploitation réelles. 

Voici à quoi ressemblerai une machine de développeur travaillant sur des stacks technologiques différentes :

![Image](/images/posts/vagrant-et-azure/image5-1.png)

Ca semble sympa non ? Voyons comment l'utiliser avec [Windows Azure](http://azure.microsoft.com/fr-fr/).

### Prérequis 

Pour la suite de cet article, vous devez :

- ouvrir un compte sur [Azure](http://azure.microsoft.com/fr-fr/) 
- installer [Vagrant](https://www.vagrantup.com/downloads.html)

### Vagrant Azure Provider & VagrantBox

Pour provisionner des machines sur Azure, il est nécessaire d'installer le plugin [Vagrant Azure Provider](https://github.com/MSOpenTech/vagrant-azure).

Depuis un terminal, lançons la commande suivante :

``` bash
$ vagrant plugin install vagrant-azure
```

Une fois le plugin installé, il faut créer ou installer une box. Dans le monde Vagrant, les box sont tout simplement des configurations d’environnement générique. 

Afin de démarrer rapidement, installons la dummy box proposée par MsOpenTech :

``` bash
$ vagrant box add azure https://github.com/msopentech/vagrant-azure/raw/master/dummy.box
```

Si vous souhaitez créer votre propre box, un exemple est disponible [ici](https://github.com/MSOpenTech/vagrant-azure/tree/master/example_box).

### VagrantFile

Créons un fichier Vagrantfile décrivant notre environnement :

``` bash
Vagrant.configure('2') do |config|
  config.vm.box = 'azure'
  config.vm.provider :azure do |azure|
    azure.vm_size = 'ExtraSmall'

    azure.mgmt_certificate = 'vagrantdemo.pem'
    azure.mgmt_endpoint = 'https://management.core.windows.net'
    azure.subscription_id = 'AZURE-SUBSCRIPTION-ID'

    azure.vm_image = 'a699494373c04fc0bc8f2bb1389d6106__Windows-Server-2012-R2-201411.01-en.us-127GB.vhd'
 
    azure.vm_user = 'mickael' 
    azure.vm_password = 'VagrantDemoP@ssw0rd' 

    azure.vm_name = 'vagrantdemo'

    azure.deployment_name = 'vagrantdemo'
    azure.vm_location = 'North Europe'

    azure.winrm_transport = [ 'http' ] 

    azure.tcp_endpoints = '80,3389:3390' 
  end

end
```

Ayant rencontré quelques problèmes lors de mes expérimentations, voici quelques points qu'il me semble intéressant de mentionner :

- **azure.subscription_id** : Le subscription id est disponible via le portail Azure dans l'onglet Paramètre, Abonnement.

- **azure.vm_image** : Les choix disponibles ne sont pas spécialement documentés... Un moyen simple de trouver le nom de l'image qui vous intéresse est d'utiliser les Cmdlets Azure comme **Get-AzureVMImage**.

- **azure.mgmt_certificate** : Le SDK Azure Ruby sur lequel se repose Vagrant Azure est sensé supporter les certificats .pem et .pfx. Neanmoins due à un [bug](https://github.com/MSOpenTech/vagrant-azure/issues/25), vous devez utiliser un .pem. Pour cela, vous trouverez les lectures suivantes utiles : [Créer un certificat de mangement pour Azure](http://msdn.microsoft.com/en-us/library/azure/gg551722.aspx) et [Exporter un pfx en pem](http://help.globalscape.com/help/eft6-2/mergedprojects/eft/exporting_a_certificate_from_pfx_to_pem.htm).

- **azure.vm_name** : Je vous conseille de donner un nom en minuscule à votre VM ou vous vous exposerez à des erreurs assez peu parlantes... Cela provient d'un autre bug du SDK Azure Ruby reporté [ici](https://github.com/MSOpenTech/vagrant-azure/issues/11).

- **azure.vm_size** : Quelques options possibles sont : ExtraSmall, Smal, Medium, Large, ExtraLarge, etc. Elles sont "documentées" [ici](https://github.com/Azure/azure-sdk-for-ruby#virtual-machine-management).

- **azure.vm_location** : Les options possibles sont West Europe, North Europe, Southeast Asia, East Asia, East US, West US et sont extraites de la source [suivante](https://github.com/Azure/azure-sdk-for-ruby/blob/9d5f660043f7fc607d644bf1659efa8985ae5b2e/test/fixtures/list_locations.xml).

Pour plus d'informations, l'ensemble des options de configurations sont disponibles [ici](https://github.com/MSOpenTech/vagrant-azure#configuration).

### Vagrant Up

Notre fichier de configuration terminé, on peut maintenant provisionner une machine avec la commande :

``` bash
$ ls
      vagrantdemo.pem
      Vagrantfile

$ vagrant up --provider=azure
```

![Image](/images/posts/vagrant-et-azure/image1-2.png)

Une fois, la commande terminée, nous nous retrouvons avec une nouvelle VM tourant sur Windows Azure.

![Image](/images/posts/vagrant-et-azure/image2-1.png)

Il est alors possible de se connecter directement à la machine en RDP avec **vagrant rdp** ou bien de la détruire avec **vagrant destroy**.

![Image](/images/posts/vagrant-et-azure/image3-1.png)

![Image](/images/posts/vagrant-et-azure/image4-1.png)

### Conclusion

Pour moi, Vagrant répond de manière simple aux besoins de partage, de reproductibilité et de versionning d'environnements. Même s'il est déjà possible de gérer ses VMs Azure depuis powershell, Vagrant possède l'avantage d'offrir une couche d'abstraction vis à vis d'un fournisseur de ressources. Il existe en effet des providers pour VMWare, Amazon, etc...

Pour finir, il est bien entendu rare de pouvoir déployer directement une application sur une machine nue. Afin d'atteindre une configuration donnée, Vagrant supporte des outils comme [Puppet](http://puppetlabs.com/), [Chef](https://www.getchef.com/chef/) et nous verrons dans un prochain article comment les mettre en place.
