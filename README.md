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
git branch -d 123_ma_feature
```


## Philosophie

Dans ce modèle, les `feature_branches` divergent `de master`. Cela permet d'avoir des branches toujours livrables, indépendamment les unes des autres. 

Naturellement, plus on attend, plus elles s'éloignent de `master`. Pour qu'elles restent toujours livrables, il faut les tenir à jour par rapport à `master`. Autrement dit, régulièrement re-merger master dans les `feature_branches` qui en dérivent.

Cela évite aussi d'avoir à résoudre plusieurs fois le même conflit dans plusieurs branches, je le fais systématiquement dans les étapes de mise en recette et mise en production.


## Mises à jour de la base de données

Les scripts de mise à jour de la base de données doivent pouvoir être appliqués automatiquement. On les nomme de manière à ce que git nous prévienne si deux scripts de mise à jour risquent de rentrer en conflit.

Ainsi, la base de données doit contenir son propre numéro de version. L'outil appliquant la mise à jour automatiquement se chargera d'incrémenter ce numéro après avoir appliqué un script de mise à jour.

Les scripts de mise à jour sont enregistrés :
- dans la feature branch qui les nécessite
- dans un dossier `database/updates` 
- dans un fichier de la forme `n+1.sql` ou `n` est le numéro de version de la base de données sur laquelle le script doit s'appliquer

Ces scripts :
- sont au format UTF-8 sans BOM, avec retours à la ligne de type unix
- commencent par `SET NAMES 'utf8';`
- ne doivent pas faire apparaître le nom de la base de donnée dans les requêtes

### Gestion des conflits

Considérons le cas où l'on a deux branches qui contiennent un script de mise à jour de base de données ayant le même nom.

Lorsqu'on va vouloir mettre en recette ces deux branches, git va lever un conflit puisque les deux scripts ont le même nom. La résolution de ce conflit __ne doit pas changer le nom du fichier__ de mise à jour. Au contraire, elle doit s'assurer d'intégrer harmonieusement les deux scripts en conflit.

Lorsqu'on mettra en production l'une de ces branches, et qu'on ira remerger master dans l'autre branche pour la tenir à jour (ce qu'on doit faire comme expliqué dans la section précédente nommée "Philosophie"), git va nous informer d'un conflit : on a sur master un fichier déjà nommé comme notre script de mise à jour. C'est à ce moment là seulement qu'on __renommera notre fichier de mise à jour en incrémentant son numéro__, après éventuelles adaptations.

#### Raffinement

On peut utiliser une branche intermédiaire entre les branches `master` et `devel` évoquées plus haut. Cela demande un peu plus de gymnastique pour jongler entre les branches, mais cela procure deux avantages :
- ne pas mettre en berne la version de recette pendant le temps où l'on travaille à la résolution des conflits entre scripts de mise à jour
- ne livrer sur la version de recette que des scripts de mise à jour prêts, ce qui permet de la faire évoluer comme ont fait évoluer la version de production. Ainsi, si on a des données de test, pas besoin de reset la base de données.

On nommera cette branche `integ`. On merge dans cette branche tout ce qui doit aller dans `devel`, et on ne merge plus rien dans `devel` hormis ce qui vient de la branche `integ`.
