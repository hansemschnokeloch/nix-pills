# Entrer dans l'environnement {#enter-environment}

Bienvenue dans la troisième Nix pill. Dans la [deuxième pill](02-install-on-your-running-system.md), nous avons installé Nix sur notre système. Nous pouvons enfin commencer à l'utiliser un peu. Ces instructions concernent également les utilisateurs de NixOS.

## Entrer dans l'environnement


**Si vous utilisez NixOS, vous pouvez passer directement à l'[étape suivante](#install-something).**

Dans l'article précédent, nous avons créé un utilisateur Nix, commençons donc par usurper son identité avec `su - nix`. Si votre `~/.profile` a été évalué, vous devriez maintenant pouvoir exécuter des commandes telles que `nix-env` et `nix-store`.

Si ce n'est pas le cas :

```console
$ source ~/.nix-profile/etc/profile.d/nix.sh
```

Pour rappel , `~/.nix-profile/etc` pointe vers la dérivation `nix-2.1.3`. À ce stade, nous sommes dans notre profil utilisateur Nix.

## Installer quelque chose {#install-something}

Enfin du concret ! L'installation dans l'environnement Nix est un processus intéressant. Installons `hello`, un outil CLI simple qui affiche `Hello world` et est principalement utilisé pour tester les compilateurs et les installations de paquets.

Retour à l'installation :

```console
$ nix-env -i hello
installing 'hello-2.10'
[...]
building '/nix/store/0vqw0ssmh6y5zj48yg34gc6macr883xk-user-environment.drv'...
created 36 symlinks in user environment
```

Vous pouvez désormais exécuter `hello`. Quelques remarques :

- Nous avons installé le logiciel en tant qu'utilisateur, et uniquement pour l'utilisateur Nix.

- Cela a créé un nouvel environnement utilisateur. C'est une nouvelle génération de notre _profile_ utilisateur Nix.

- L'outil [nix-env](https://nix.dev/manual/nix/stable/command-ref/nix-env) gère les environnements, les _profiles_ et leurs générations.

- Nous avons installé `hello` en utilisant le nom de la dérivation sans la version. Je répète : nous avons spécifié le **nom de la dérivation** (mais pas la version) pour l'installer.

Nous pouvons lister les générations sans parcourir l'arborescence de `/nix` :

```console
$ nix-env --list-generations
    1   2014-07-24 09:23:30
    2   2014-07-25 08:45:01   (current)
```

Listons les dérivations installées :

```console
$ nix-env -q
nix-2.1.3
hello-2.10
```

Alors, où `hello` a-t-il réellement été installé ? `which hello` nous indique `~/.nix-profile/bin/hello` qui pointe vers le store. Nous pouvons également lister les chemins de dérivation avec `nix-env -q --out-path`.
C'est donc ainsi qu'on nomme ces chemins de la dérivation : la **sortie** d'un _build_.

## Fusion de chemins

Vous voulez à présent sans doute lancer `man` pour obtenir de la documentation. Même si vous disposez déjà de `man` à l'échelle du système en dehors de l'environnement Nix, vous pouvez l'installer et l'utiliser dans Nix avec `nix-env -i man-db`.
Comme précédemment, cela va créer une nouvelle génération, et `~/.nix-profile` va pointer vers elle.

Examinons un peu le [_profile_](https://nix.dev/manual/nix/stable/package-management/profiles) :

```console
$ ls -l ~/.nix-profile/
dr-xr-xr-x 2 nix nix 4096 Jan  1  1970 bin
lrwxrwxrwx 1 nix nix   55 Jan  1  1970 etc -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/etc
[...]
```

Voilà qui est intéressant. Lorsque seul `nix-2.1.3` était installé, `bin` était un lien symbolique vers `nix-2.1.3`. Maintenant que nous avons installé autre chose (`man`, `hello`), ce n'est plus un lien symbolique mais un répertoire.

```console
$ ls -l ~/.nix-profile/bin/
[...]
man -> /nix/store/83cn9ing5sc6644h50dqzzfxcs07r2jn-man-1.6g/bin/man
[...]
nix-env -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env
[...]
hello -> /nix/store/58r35bqb4f3lxbnbabq718svq9i2pda3-hello-2.10/bin/hello
[...]
```

C'est plus clair maintenant. `nix-env` a fusionné les chemins des dérivations installées. `which man` pointe vers le profil Nix, plutôt que vers le `man` du système, car `~/.nix-profile/bin` se trouve en tête de `$PATH`.

## Retour en arrière et changement de génération

La dernière commande a installé `man`. Nous devons être à la génération 3, sauf si vous avez changé quelque chose entre-temps. Disons que nous voulons revenir à la précédente génération :

```console
$ nix-env --rollback
switching from generation 3 to 2
```

Désormais, `nix-env -q` ne liste plus `man`. La commande `` ls -l `which man` `` devrait maintenant pointer vers la copie système.

Trêve de retour en arrière, revenons à la génération la plus récente :

```console
$ nix-env -G 3
switching from generation 2 to 3
```

Je vous invite à lire la page de manuel de `nix-env`. `nix-env` nécessite une opération à effectuer, il y a des options communes à toutes les opérations, ainsi que des options spécifiques à chaque opération.

Vous pouvez bien évidemment [désinstaller](https://nix.dev/manual/nix/stable/command-ref/nix-env/uninstall) et [mettre à niveau](https://nix.dev/manual/nix/stable/command-ref/nix-env/upgrade) des paquets.

## Interroger le store

Jusqu'à présent, nous avons appris à interroger et manipuler l'environnement. Mais tous les composants de l'environnement pointent vers le _store_.

Pour interroger et manipuler le _store_, il y a la commande `nix-store`. Nous pouvons faire des choses intéressantes, mais nous ne verrons pour l'instant que quelques requêtes.

Pour afficher les dépendances d'exécution directes de `hello` :

```console
$ nix-store -q --references `which hello`
/nix/store/fg4yq8i8wd08xg3fy58l6q73cjy8hjr2-glibc-2.27
/nix/store/58r35bqb4f3lxbnbabq718svq9i2pda3-hello-2.10
```

L'argument de `nix-store` peut être n'importe quoi tant qu'il pointe vers le _Nix store_. Il va suivre les liens symboliques.

Cela peut ne pas vous paraître sensé pour l'instant, mais affichons les dépendences inverses de `hello` :

```console
$ nix-store -q --referrers `which hello`
/nix/store/58r35bqb4f3lxbnbabq718svq9i2pda3-hello-2.10
/nix/store/fhvy2550cpmjgcjcx5rzz328i0kfv3z3-env-manifest.nix
/nix/store/yzdk0xvr0b8dcwhi2nns6d75k2ha5208-env-manifest.nix
/nix/store/mp987abm20c70pl8p31ljw1r5by4xwfw-user-environment
/nix/store/ppr3qbq7fk2m2pa49i2z3i32cvfhsv7p-user-environment
```

Est-ce ce à quoi vous vous attendiez ? Il s'avère que nos environnements dépendent de `hello`. Oui, cela signifie que les environnements sont dans le _store_, et comme ils contiennent des liens symboliques vers `hello`, l'environnement dépend donc de `hello`.

Deux environnements ont été listés, la génération 2 et la génération 3, car ce sont ceux qui avaient `hello` installé.

Le fichier `manifest.nix` contient des métadonnées sur l'environnement, telles que les dérivations installées. Ainsi, `nix-env` peut les lister, les mettre à jour ou les supprimer. Et à nouveau, le `manifest.nix` actuel se trouve dans `~/.nix-profile/manifest.nix`.

## Fermetures (_closures_)

La [_closure_](https://fr.wikipedia.org/wiki/Fermeture_(informatique)) d'une dérivation est une liste récursive de toutes ses dépendances, incluant absolument tout ce qui est nécessaire pour utiliser cette dérivation.

```console
$ nix-store -qR `which man`
[...]
```

Copier toutes ces dérivations dans le _Nix store_ d'une autre machine vous permet d'exécuter `man` immédiatement sur cette autre machine. C'est la base du déploiement avec Nix, et vous pouvez déjà entrevoir le potentiel lors du déploiement de logiciels dans le cloud (indice : `nix-copy-closures` et `nix-store --export`).


Une vue plus agréable de la _closure_ :

```console
$ nix-store -q --tree `which man`
[...]
```

Avec la commande ci-dessus, vous pouvez découvrir exactement pourquoi une dépendance de l'environment d'exécution, qu'elle soit directe ou indirecte, existe pour une dérivation donnée.

Il en va de même pour les environnements. À titre d'exercice, exécutez `nix-store -q --tree ~/.nix-profile`, et constatez que les premiers enfants sont des dépendances directes de l'environnement utilisateur : les dérivations installées, et `manifest.nix`.

## Résolution des dépendances

Il n'existe pas d'outil comme `apt` qui résout un [problème SAT](https://fr.wikipedia.org/wiki/Probl%C3%A8me_SAT) afin de satisfaire les dépendances avec des bornes inférieures et supérieures sur les versions. Cela n'est pas nécessaire car toutes les dépendances sont statiques : si une dérivation X dépend d'une dérivation Y, alors elle en dépend toujours. Une version de X qui dépendrait de Z serait une dérivation différente.

## Restaurer à la dure

```console
$ nix-env -e '*'
uninstalling 'hello-2.10'
uninstalling 'nix-2.1.3'
[...]
```

Oups, cela a désinstallé toutes les dérivations de l'environnement, y compris Nix. Cela signifie que nous ne pouvons même pas exécuter `nix-env`, que faire maintenant ?

Auparavant, nous obtenions `nix-env` depuis l'environnement. Les environnements sont une commodité pour l'utilisateur, mais Nix est toujours là dans le store !

Tout d'abord, choisissez une dérivation `nix-2.1.3` : `ls /nix/store/*nix-2.1.3`, disons `/nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3`.

La première option est de revenir en arrière :

```console
$ /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env --rollback
```

La seconde option est d'installer Nix, ce qui va créer une nouvelle génération :

```console
$ /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env -i /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env
```

## Canaux (_Channels_)

D'où viennent les paquets ? Nous en avons déjà parlé dans le [deuxième article](02-install-on-your-running-system.md). Il existe une liste de _channels_ à partir desquels nous obtenons les paquets, bien que nous utilisions généralement un seul _channel_. L'outil pour gérer les _channels_ est [nix-channel](https://nix.dev/manual/nix/stable/command-ref/nix-channel).

```console
$ nix-channel --list
nixpkgs http://nixos.org/channels/nixpkgs-unstable
```
Si vous utilisez NixOS, vous ne verrez peut-être aucune sortie de la commande ci-dessus (si vous utilisez le _channel_ par défaut), ou vous verrez peut-être un _channel_ dont le nom commence par "nixos-" au lieu de "nixpkgs".

C'est essentiellement le contenu de `~/.nix-channels`.

<div class="info">

Remarque : `~/.nix-channels` n'est pas un lien symbolique vers le _Nix store_ !

</div>

Pour mettre à jour le _channel_, exécutez `nix-channel --update`. Cela téléchargera les nouvelles expressions Nix (descriptions des paquets), créera une nouvelle génération du _profile_ des _channels_ et le décompressera dans `~/.nix-defexpr/channels`.

C'est assez similaire à `apt-get update`. (Voir [ce tableau](https://wiki.nixos.org/wiki/Cheatsheet) pour une correspondance approximative entre la gestion des paquets d'Ubuntu et de NixOS.)

## Conclusion

Nous avons appris à interroger et manipuler l'environnement utilisateur en installant et désinstallant des logiciels. La mise à jour des logiciels est très simple, comme décrit dans [le manuel](https://nix.dev/manual/nix/stable/command-ref/nix-env/upgrade) (`nix-env -u` mettra à jour tous les paquets de l'environnement).

Chaque fois que nous modifions l'environnement, une nouvelle génération est créée. Le passage d'une génération à une autre est simple et immédiat.

Enfin, nous avons appris à interroger le _store_. Nous avons examiné les dépendances et les dépendances inverses des chemins du _store_.

Nous avons vu comment les liens symboliques sont utilisés pour composer des chemins à partir du _Nix store_, une astuce utile.

Une analogie rapide avec les langages de programmation : vous avez le tas avec tous les objets, qui correspond au _Nix store_. Vous avez des objets qui pointent vers d'autres objets, ceux-ci correspondent aux dérivations. C'est une métaphore suggestive, est-ce la bonne voie à suivre ?

## Next pill

...[Next pill: Les bases du langage Nix](04-nix-language-basics.md) nous apprendra les bases du langage Nix. Le langage Nix est utilisé pour décrire comment construire des dérivations, et c'est la base de tout le reste, y compris NixOS. Il est donc très important de comprendre à la fois la syntaxe et la sémantique du langage.
