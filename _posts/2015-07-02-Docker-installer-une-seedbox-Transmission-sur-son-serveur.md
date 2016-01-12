---
layout: post
title: "Docker : installer une seedbox (Transmission) sur son serveur"
date: 2015-07-02 13:54:02 +02:00
tags:
    - Linux
    - FTP
    - Docker
    - Transmission
description: "Mise en place d'une seedbox sur son serveur dédié (avec Transmission). Utilisation de Docker qui permet d'automatiser l'installation et le déploiement de toutes nos applications (Transmission, Apache/Nginx, Pure-FTPD)."
disqus_identifier: 31
---

## Introduction

Cet article a pour but d'expliquer comment mettre en place une seedbox sur son serveur dédié, ainsi que les outils nécessaires afin de pouvoir récupérer les fichiers téléchargés. Il utilisera :

- *Transmission*, un client *BitTorrent*.
- *Nginx* (ou *Apache*), un serveur web, afin de pouvoir parcourir et récupérer les fichiers téléchargés.
- *Pure-FTPD* (ou directement en *sFTP*), afin de pouvoir parcourir et récupérer les fichiers téléchargés via le protocole *FTP* (ou *sFTP*), qui est plus pratique à utiliser que le navigateur lorsqu'il y a beaucoup de fichiers.
- *Docker*, qui permet d'automatiser l'installation et le déploiement de nos applications (*Transmission*, *Nginx* / *Apache*, *Pure-FTPD*).


## Docker

*Docker* vous permet d'empaqueter une application avec toutes ses dépendances dans une unité standardisée pour le développement de logiciels.

Chaque application est désignée par un *container*, qui est semblable à une Machine Virtuelle (isolement, et allocation des ressources), mais avec une architecture différente qui lui permet d'être bien plus portable et efficace.
Contrairement à une *VM*, les *containers* utiliseront *l'OS* de la machine hôte. Si vous souhaitez savoir un peu plus comment fonctionne *Docker* : [What is Docker?][what-is-docker].

## Préambule

Nous allons tout d'abord préparer notre linux, créer un utilisateur pour notre seedbox, installer *Docker*, créer les dossiers, ...

### Installation de docker

L'installation de *Docker* peut, pour la plupart du temps, se faire directement depuis les dépôts de votre distribution. Sur *Ubuntu*, ce sera :
{% highlight console linenos %}
#> apt-get install docker.io
{% endhighlight %}

Si jamais vous ne trouvez pas le packet (ou qu'il n'est tout simplement pas présent sur vos dépôts) :
{% highlight console linenos %}
#> wget -qO- https://get.docker.com/ | sh
{% endhighlight %}

### Préparation des répertoires

Créez ensuite un utilisateur `seedbox`, et ajoutez le dans le groupe `docker` (il aura ainsi les droits nécessaires pour gérer les différents *containers* et *images* de *Docker*) :
{% highlight console linenos %}
#> useradd seedbox -m
#> passwd seedbox
#> gpasswd -a seedbox docker
{% endhighlight %}

J'ai choisis de mettre les fichiers téléchargés dans le dossier `/home/seedbox/downloads`. Libre à vous de choisir ce que vous préférez, mais n'oubliez pas de changer en conséquence pour les prochaines commandes. Pensez également à autoriser l'écriture dans le dossier pour tout le monde.
{% highlight console linenos %}
#> su seedbox
$> mkdir /home/seedbox/downloads
$> chmod 777 -R /home/seedbox/downloads
{% endhighlight %}

## Contrainer Transmission

Démarrer *Docker* si ce n'est pas déjà fait :
{% highlight console linenos %}
#> /etc/init.d/docker.io start
{% endhighlight %}

Nous allons utiliser une *image Docker* déjà créée pour *Transmission* : [stevenmartins/docker-transmission][stevenmartins-docker-transmission].

Pour récupérer cette image sur votre serveur :
{% highlight console linenos %}
$> docker pull stevenmartins/docker-transmission
{% endhighlight %}

Puis pour exécuter le *container* :
{% highlight console linenos %}
$> docker run -d -p 51413:51413 -p 51413:51413/udp -p 2000:9091 -e TRANSMISSION_PASS=password -v /home/seedbox/downloads:/transmission/downloads --name seedbox stevenmartins/docker-transmission
{% endhighlight %}

Quelques détails sur les paramètres cette commande `docker run` :

- `-d` : permet de le lancer en *daemon* (en tâche de fond)
- `-p 51413:51413` : par défaut, le *container* est cloisonné, il ne peut pas communiquer avec la machine hôte, et donc avec l'extérieur. L'option `p` permet justement d'autoriser certaines communications, en reliant des ports. Le premier port est celui de la machine hôte, le second est celui à l'intérieur du *container*. *Transmission* utilise le port 51413 pour télécharger les torrents (par défaut) et a donc besoin que ce port soit ouvert depuis l'extérieur. Ainsi, tout ce qui se passe sur le port 51413 du serveur sera redirigé vers le port 51413 du *container*.
- `-p 51413:51413/udp` : Même option, mais pour accepter également les connexions *UDP*.
- `-p 2000:9091` : l'interface web de *Transmission* se lance sur le port 9091. Si vous souhaitez accéder à l'interface depuis votre serveur, vous aurez besoin de relier un port de votre choix vers celui-ci. Rien ne vous oblige à utiliser le port 9091 ; j'ai choisi le port 2000. Ainsi, en allant sur `xxx.xxx.xxx.xxx:2000`, j’accéderai à l'interface web du client *BitTorrent*.
- `-e TRANSMISSION_PASS=password` : l'option `e` permet de définir une variable d'environnement. Cette variable est utilisée par le script de démarrage du *Dockerfile* de l'image choisie (`stevenmartins/docker-transmission`).
- `-v /home/seedbox/downloads:/transmission/downloads` : c'est le même fonctionnement que pour l'option port, mais pour partager cette fois ci des dossiers entre la machine hôte et le *container*. *Transmission* sauvegardant les fichiers téléchargés dans `/transmission/downloads`, nous linkons ce dossier vers `/home/seedbox/downloads` afin qu'ils soient en fait stockés sur notre machine hôte.
- `--name seedbox` : nom pour notre *container*.
- `stevenmartins/docker-transmission` : le nom de l'image *Docker*.


Une fois cette commande exécutée, vous pouvez donc accéder à votre interface *Transmission* via : `http://xxx.xxx.xxx.xxx:2000/`.
L'identifiant est `transmission` et le mot de passe celui que vous avez envoyé dans la commande précédente, via la variable `TRANSMISSION_PASS`.

## Container Nginx

Nous allons ensuite mettre en place un serveur web pour accéder à notre dossier `/home/seedbox/downloads` directement depuis le navigateur. Nous allons utiliser *Nginx*, mais il vous est aussi possible de prendre à la place *Apache*, il vous suffira de choisir la bonne image sur le [Hub de Docker?][hub-de-docker].

{% highlight console linenos %}
$> docker pull nginx
{% endhighlight %}

Avant de lancer notre *container*, nous allons devoir créer quelques fichiers de configurations pour qu'il puisse fonctionner correctement.

Créez le fichier `/home/seedbox/nginx-conf/.htpasswd` et mettez les identifiants que vous souhaitez utiliser pour accéder au serveur web :
{% highlight properties linenos %}
transmission:$1$.YDNjAwf$BK6OHV8.0XNIKgGpx2CY4.
{% endhighlight %}
Comme vous le voyez, le mot de passe est chiffré (ici le mot de passe est `seedbox`). Vous pouvez utiliser des outils en ligne pour générer le vôtre, ou le hasher vous même (en *MD5*, en *BlowFish*, ...).

Créez le fichier `/home/seedbox/nginx-conf/seedbox.conf` :
{% highlight nginx linenos %}
server {
        listen 80;
        root /downloads;
        server_name 127.0.0.1;
        location / {
                autoindex on;
                auth_basic "Restricted";
                auth_basic_user_file /etc/nginx/conf.d/.htpasswd;
        }
}
{% endhighlight %}

C'est le fichier de configuration pour *Nginx*. Il affichera les fichiers pour le dossier `/downloads`, écoutera sur le port 80 du *container* et demandera une authentification grâce au fichier `/etc/nginx/conf.d/.htpasswd`.

Il ne nous reste donc plus qu'à relier tous ces paramètres avec notre machine hôte lors du lancement :
{% highlight console linenos %}
$> docker run -p 2001:80 -d -v /home/seedbox/downloads:/downloads:ro -v /home/seedbox/nginx-conf:/etc/nginx/conf.d:ro --name seedbox_nginx nginx
{% endhighlight %}

- **-p 2001:80**, le port 2001 permettra d'accéder au *Nginx* du *container* (puisqu'il écoute sur le port 80).
- **-d**, en mode *daemon*.
- **-v /home/seedbox/downloads:/downloads:ro**, permet à *Nginx* d'accéder aux fichiers téléchargés. L'option `ro` en plus signifie `read only`, l'interdisant ainsi d'écrire dans le dossier (il n'a pas besoin de pouvoir le faire, il ne sert qu'à télécharger les fichiers).
- **-v /home/seedbox/nginx-conf:/etc/nginx/conf.d:ro**, permet d'ajouter nos fichiers de configurations créés précédemment directement sur le *container*.
- **--name seedbox_nginx**, le nom de l'instance.
- **nginx**, l'image de *Docker*.


`http://xxx.xxx.xxx.xxx:2001/`
(avec l'identifiant que vous avez renseigné dans le `.htpasswd`, soit `transmission / seedbox` dans l'exemple)

## Accès FTP

Même si vous pouvez déjà récupérer vos fichiers depuis le navigateur, c'est parfois pas très pratique lorsque vous avez beaucoup de fichiers à télécharger.
Il est plus simple d'utiliser le protocole *FTP* dans ce cas, avec le logiciel *FileZilla* par exemple.

Vous pourriez normalement déjà pouvoir y accéder, en *sFTP* (*FTP* en passant par la connexion *SSH*) :

- Hôte : `xxx.xxx.xxx.xxx`
- Username : `seedbox`
- Password : `votre mot de passe choisi pour ce compte dans la première partie`
- Port : 22 (le port *SSH*, et non le port par défaut, 21, qui est le *FTP*)


Si vous souhaitez tout de même installer un vrai serveur *FTP*, vous pouvez aussi le faire avec une image *Docker* déjà créée (avec *pure-FTPD*) :
{% highlight console linenos %}
$> docker pull stilliard/pure-ftpd
$> docker run -d -p 9002:21 -v /home/seedbox/downloads:/downloads:ro --name seedbox_ftp stilliard/pure-ftpd
{% endhighlight %}

Pour ajouter un nouvel utilisateur, vous aurez besoin de vous connecter au *container* :
{% highlight console linenos %}
$> docker exec -it seedbox_ftp /bin/bash
seedbox_ftp> pure-pw useradd seedbox -u ftpuser -d /downloads
seedbox_ftp> pure-pw mkdb
{% endhighlight %}

[what-is-docker]: https://www.docker.com/whatisdocker
[hub-de-docker]: https://registry.hub.docker.com
[stevenmartins-docker-transmission]: https://registry.hub.docker.com/u/stevenmartins/docker-transmission/
