# Modèle d'utilisation de Git

Modèle d'utilisation de Git pour un workflow avec versions de développement, recette, et production.

__Attention__ : sauf mention contraire, les lignes de commandes ci-dessous sont à exécuter sur les versions des développeurs (donc jamais sur la version de recette et encore moins la version de production !)


## Développement dans des branches

Le code est versionné en utilisant git en suivant le modèle suivant : 
- la branche `master` correspond à la version de production
- la branche `devel` correspond à la version de recette
- le développement s'effectue dans des `feature_branches` : ce sont des branches qui divergent de `master`. 

On crée généralement une branche par ticket. Dans l'exemple ci-dessous, __123__ correspond au numéro du ticket auquel correspond la branche.

__Création d'une `feature_branche`__
```bash
git checkout master
git pull
git checkout -b 123_ma_feature
git push -u origin 123_ma_feature
```

## Mise en recette

Une fois le développement terminé on les met en recette. Pour ce faire, on commence par s'assurer que notre `feature_branch` est à jour par rapport à `master` (pour ne pas merger du code obsolète, qui génèrerait des conflits), puis on merge notre `feature_branch` dans `devel`.

```bash
# met à jour master
git checkout master
git pull

# met à jour 123_ma_feature par rapport à master
git checkout 123_ma_feature
git merge master

# met à jour la recette et met en recette 123_ma_feature
git checkout devel
git pull
git merge 123_ma_feature
git push
```


## Mise en production

Pour mettre en production, on merge notre `feature_branch` dans `master`.

```bash
git checkout master
git pull
git merge 123_ma_feature
git push
```


### Contrôle après mise en production

Après une mise en production, il faut vérifier que `devel` est le plus à jour possible par rapport à `master`, pour s'assurer qu'on n'a pas laissé des conflits à résoudre qui pourraient retomber sur les autres membres de l'équipe : 

```bash
git checkout master && git pull
git checkout devel && git pull
git merge master
```

Autre méthode (à vérifier tout de même) : 

```bash
# vérifie s'il y a des choses dans master à merger dans devel. 
# doit renvoyer : 0
git format-patch $(git merge-base devel master)..master --stdout | wc -l
```

Il ne devrait pas y avoir de nouveautés à merger à cette étape. S'il y en a c'est que quelqu'un utilise mal le workflow.


### Avertissement

__Attention__ : à aucun moment il ne devrait être nécessaire de merger `devel` (ou une branche qui en divergerait) dans notre `feature_branch` ! Il y a risque de mettre en production la version de recette sinon !

On peut toutefois le faire consciemment, à condition que la branche `devel` ait bien été stabilisée, pour assurer d'éventuels points de synchronisation. On pourrait alors détruire `devel` et la recréer, toujours en divergeant de `master`.


### Ménage

Une fois qu'on a mis notre `feature_branch` en production, on peut la supprimer :
```bash
git branch -d feature_branch
```


## Philosophie

Dans ce modèle, les `feature_branches` divergent `de master`. Cela permet d'avoir des branches toujours livrables, indépendamment les unes des autres. 

Naturellement, plus on attend, plus elles s'éloignent de `master`. Pour qu'elles restent toujours livrables, il faut les tenir à jour par rapport à `master`. Autrement dit, régulièrement re-merger master dans les `feature_branches` qui en dérivent.

Cela évite aussi d'avoir à résoudre plusieurs fois le même conflit dans plusieurs branches, je le fais systématiquement dans les étapes de mise en recette et mise en production.


## Mises à jour de la base de données

- on les enregistre dans un dossier database/updates 
- on les enregistre dans un fichier de la forme `YYYYmmdd-HHMM-libelle.sql`
- on les enregistre dans la feature branch qui les implique
- le fichier doit être au format UTF-8 sans BOM, avec retours à la ligne de type unix
- le fichier doit commencer par `SET NAMES 'utf8';`
- il ne faut pas faire apparaître le nom de la base de donnée dans les requêtes

Elles devront pouvoir être appliquées à l'aide de la ligne de commande suivante : 

```bash
mysql nom_de_la_base --show-warnings < database/updates/YYYYmmdd-HHMM-libelle.sql > database/updates/YYYYmmdd-HHMM-libelle.log
```



### Améliorations possibles

Les scripts de mise à jour de la base de données devraient pouvoir être appliqués automatiquement, avec rollback automatique en cas de problème. C'est l'objet du ticket https://github.com/B3Software/sud-convergences/issues/443 qui n'est hélas pas vraiment applicable tant que la base de données sera aussi lourde.

Réflexion en cours pour un outil permettant de gérer les upgrades de la base de données avec les branches de développement.

L'idée est d'enregistrer en base un numéro de version `numversion` de la base de données. Ce numéro sera incrémenté à chaque application d'un script. On pourra aussi enregistrer la liste des scripts d'upgrade appliqués ainsi que la date d'application pour faciliter le suivi.

- On nommera les scripts de migration ainsi : upgrade-from-numversion.php 
- Pour savoir si on peut appliquer un script il suffira de vérifier si la base sur laquelle on veut l'appliquer est bien en version numversion. Si ce n'est pas le cas on refuse d'appliquer le script (ou la série de scripts l'incluant).
- Si 2 scripts d'upgrade sont nommés identiquement upgrade-from-numversion.php, Git levera un conflit lors du merge de branches, ce qui alertera le développeur qu'il faut vérifier l'ordre d'application de ces 2 scripts. 

En cas de conflit ou de refus d'appliquer un script, il faudra mettre à jour le script (au moins en le renommant en upgrade-from-numversion+1.php par exemple).

Pour aller plus loin on pourra : 

- lorsqu'on détecte un nouveau scripts, alerter si son numversion est < numversion de la base de données
- lorsqu'on détecte un nouveau scripts, alerter si son numversion est > numversion de la base de données 
- utiliser les hooks de Git pour refuser un commit sur `master` qui impliquerait d'appliquer des scripts si ils ne sont pas en cohérence avec la base de données.
