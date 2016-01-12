---
layout: post
title: "Migration de votre système de commentaires vers Disqus"
date: 2014-08-12 13:54:02 +02:00
tags:
    - Python
    - WXR
    - Disqus
description: "Explication des manipulations pour remplacer votre propre système de commentaires vers l'outil Disqus, tout en gardant l'ensemble des commentaires de vos visiteurs."
disqus_identifier: 30
---

## Introduction

J'utilisais mon propre système de commentaires jusqu'à maintenant. J'ai décidé de le remplacer par [*Disqus*](https://disqus.com), un outil spécialisé pour ça, qui est utilisé sur pas mal de blogs.
C'est d'ailleurs un plus pour le visiteur, puisqu'il a un seul compte pour tous ses blogs compatibles.
L'administration est aussi assez bien réalisée, il est par exemple possible d'approuver des commentaires directement par *email* ou *SMS*.
Seulement voilà, je ne voulais pas non plus perdre tous mes anciens commentaires, il a donc fallu les importer sur *Disqus*.


## Migration des données sur Disqus

*Disqus* n'accepte comme forme générique qu'un format d'importation *XML* basé sur le schéma *WXR* (WordPress eXtended Rss).
C'est un format qui contient la liste de vos articles et des commentaires qui y sont associés.


Pour chaque article, vous devez renseigner :

- Son identifiant
- Son titre
- Le lien complet vers l'article
- Sa date de publication
- Son statut (actif ou non)
- Et bien sûr, ses commentaires

Pour chaque commentaire :

- Son identifiant
- Des informations sur l'auteur (nom, site internet, email, IP)
- Sa date de publication
- Son contenu
- S'il est approuvé ou non
- Et l'identifiant du commentaire parent, si votre système gérait l'imbrication de vos commentaires

Vous trouverez un exemple du format de retour sur la documentation de *Disqus* : [Custom XML Import Format](https://help.disqus.com/customer/portal/articles/472150-custom-xml-import-format).


## Script d'exportation

Même si ce format est assez simple, ce n'est quelque chose que l'on complète à la main.
J'ai donc créé un petit code *Python* pour exporter ces données d'une base de données.
Vous pouvez récupérer la dernière version sur mon *GitHub* : [github.com/Dewep/ExportWXR](https://github.com/Dewep/ExportWXR).

### Détails du dépôt

Vous avez besoin d'avoir *Python3* d'installer sur votre machine. Deux fichiers sont utiles :

- `ExportWXR.py` : c'est la classe qui permet de générer votre fichier *WXR*, vous n'êtes normalement pas censé le modifier.
- `main.py` : c'est ce fichier que vous allez devoir personnaliser en fonction du schéma de votre base de données. Une fois exécuté, il s'occupera ensuite d'appeler la classe *ExportWXR*.


### Personnalisation de main.py

Vous avez 2 méthodes dans ce fichier à adapter en fonction de votre base de données : `get_articles` (doit récupérer et retourner la liste de vos articles) et `get_comments` (doit récupérer et retourner la liste des commentaires de l'article passé en paramètre).


Même si vous ne savez pas faire de Python, vous devriez réussir à la personnaliser assez facilement, c'est juste une requête *SQL* à modifier.


Pensez également à modifier vos accès à la base de données, présents au début du fichier, dans le constructeur (`__init__`) de la classe.

### Exécution du script

Il vous suffit ensuite de lancer votre script :
{% highlight console linenos %}
$> python main.py
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" ......
{% endhighlight %}

Si vous n'avez pas d'erreurs Python et que le résultat semble correct, alors enregistrez le dans un fichier :
{% highlight console linenos %}
$> python main.py > wxr.xml
{% endhighlight %}


## Installation de Disqus

Il ne vous reste maintenant plus qu'à importer votre fichier `wxr.xml` sur votre compte *Disqus* : **https://`myblog`.disqus.com/admin/discussions/import/platform/generic/**.


Attendez une petite minute et vous devriez voir le résultat de l'importation.
Il se peut que vous rencontriez une erreur, comme par exemple, un commentaire parent qui n'existe pas, une adresse email invalide, ... l'importation aura alors échouée, vous devrez modifier vos données dans votre base de données avant de regénérer un nouveau fichier *WXR*.


Il ne reste plus qu'à installer *Disqus* sur votre site : **https://`myblog`.disqus.com/admin/settings/install/**.
