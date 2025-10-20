# Pourquoi vous devriez l'essayer

## Introduction

Bienvenue dans le premier article de la série "[Nix](https://nixos.org/nix) in pills". Nix est un gestionnaire de paquets et un système de déploiement purement fonctionnel pour POSIX.

Il y a beaucoup de documentation qui décrit ce qu'est Nix, [NixOS](https://nixos.org/nixos) et les projets connexes. Mais le but de cet article est de vous convaincre d'essayer Nix. L'installation de NixOS n'est pas obligatoire, mais je fait parfois référence à NixOS en tant qu'exemple concret d'utilisation de Nix pour construire un système d'exploitation complet.

## Raison d'être de cette série d'articles

Les manuels [Nix](https://nixos.org/manual/nix), [Nixpkgs](https://nixos.org/manual/nixpkgs/) et [NixOS](https://nixos.org/manual/nixos/) ainsi que [le wiki](https://wiki.nixos.org/) sont d'excellentes ressources pour expliquer comment Nix/NixOS fonctionne, comment vous pouvez l'utiliser, et comment des trucs sympas sont fait avec. Toutefois au départ, il se peut que certains mécanismes qui sont exécutés en arrière-plan soient difficiles à appréhender.

Cette série d'articles a pour objectif d'apporter des explications complémentaires aux documentations plus formelles.

Ce qui suit est une description de Nix. Comme pour les autres articles de cette série ("pills"), j'essaierai d'être aussi concis que possible.

## Ne pas être purement fonctionnel

La plupart, si ce n'est tous, des gestionnaires de paquets largement utilisés ([dpkg](https://wiki.debian.org/dpkg), [rpm](http://www.rpm.org/), ...) modifient l'état global du système. Si un paquet `foo-1.0` installe un programme dans `/usr/bin/foo`, vous ne pouvez pas installer `foo-1.1` en plus, à moins de modifier les chemins d'installation ou le nom de l'exécutable. Mais modifier le nom des exécutables signifie rompre la compatibilité avec les utilisateurs de cet exécutable.

Il y a quelques tentatives pour atténuer ce problème. Debian, par exemple, résout partiellement le problème avec le système [Debian alternatives](https://wiki.debian.org/DebianAlternatives).

Il est donc en théorie possible, avec certains systèmes actuels, d'installer plusieurs versions d'un même paquet, mais en pratique c'est très pénible.

Supposons que vous avez besoin d'un service nginx et d'un service nginx-openresty. Vous devez créer un nouveau paquet qui modifie tous les chemins pour avoir, par exemple, un suffixe `-openresty`.

Ou encore, supposons que vous voulez exécuter deux instances distinctes de mysql: 5.2 et 5.5. Le même principe s'applique, et vous devez également vous assurer que les deux bibliothèques mysqlclient ne rentrent pas en conflit.

Cela n'est pas impossible mais c'est _très_ contraignant. Si vous voulez installer deux stack logiciels complets tels que GNOME 3.10 et GNOME 3.12, vous pouvez imaginer la charge de travail.

Du point de vue d'un administrateur vous pouvez utiliser des conteneurs. La solution typique de nos jours est de créer un conteneur par service, surtout lorsque différentes versions sont nécessaires. Cette approche résout en partie le problème, mais à un autre niveau et avec d'autres inconvénients. Par exemple, la nécessité d'avoir des d'outils d'orchestration, la mise en place d'un cache de paquets partagé, et de nouvelles machines à surveiller plutôt que de simples services.

Du point de vue d'un développeur, vous pouvez utiliser virtualenv pour python, ou jhbuild pour gnome, ou autre chose. Mais comment faites-vous pour mélanger les deux stacks ? Comment évitez-vous de recompiler ce qui pourrait être partagé ? Vous devez également configurer vos outils de développement pour qu'ils pointent vers les différents répertoires où les bibliothèques sont installées. Sans omettre le risque que les logiciels peuvent utiliser de manière erronée des bibliothèques système.

Et ainsi de suite. Nix résout tout cela au niveau du packaging et il le fait bien. Un seul outil pour les gérer tous.

## Être purement fonctionnel


Nix ne fait pas d'hypothèse sur l'état global du système. Cela a plusieurs avantages, mais aussi quelques inconvénients bien sûr. Le cœur d'un système Nix est le _Nix store_, généralement installé sous `/nix/store`, ainsi que quelques outils pour manipuler le store. Dans Nix, il y a la notion de _derivation_ au lieu de paquet. La différence peut être subtile au début, donc j'emploierai souvent ces mots de manière interchangeable.

Les dérivations/paquets sont stockés dans le _Nix store_ comme suit : `/nix/store/«hash-name»`, où _hash_ identifie de manière unique la dérivation (c'est en réalité un peu plus complexe), et _name_ est le nom de la dérivation.

Prenons comme exemple une dérivation de bash : `/nix/store/s4zia7hhqkin1di0f187b79sa2srhv6k-bash-4.2-p45/`. C'est un répertoire dans le _Nix store_ qui contient `bin/bash`.

Cela signifie qu'il n'y a pas de `/bin/bash`, il n'y a qu'un résultat de compilation autonome dans le store. Il en va de même pour coreutils et tout le reste. Pour les rendre pratiques à utiliser depuis le shell, Nix fait en sorte que les exécutables apparaissent dans votre `PATH` comme il se doit.

Ce que nous avons est essentiellement un _store_ de tous les paquets (avec différentes versions à des emplacements différents), et dans le _Nix store_ tout est immuable.

En réalité, il n'y a pas non plus de cache ldconfig. Alors où bash trouve-t-il libc ?

```console
$ ldd `which bash`
libc.so.6 => /nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19/lib/libc.so.6 (0x00007f0248cce000)
```
Il s'avère que lorsque bash a été compilé, il l'a été par rapport à cette version spécifique de glibc dans le _Nix store_, et lors de l'exécution il aura spécifiquement besoin de cette version de glibc.

Ne vous laissez pas tromper par la version dans le nom de la dérivation : il s'agit seulement d'un nom pour faciliter la lecture. Vous pouvez avoir deux dérivations portant le même nom mais avec des hash différents : c'est le hash qui importe réellement.

Que signifie tout cela ? Cela signifie que vous pourriez exécuter mysql 5.2 avec glibc-2.18, et mysql 5.5 avec glibc-2.19. Vous pourriez utiliser votre module python avec python 2.7 compilé avec gcc 4.6 et le même module python avec python 3 compilé avec gcc 4.8, le tout dans le même système.

En d'autres termes : pas d'enfer des dépendances, ni même un algorithme de résolution des dépendances. Des dépendances directes de dérivations vers d'autres dérivations.

Du point de vue d'un administrateur : si vous avez besoin d'une ancienne version de PHP pour une application spécifique, tout en souhaitant mettre à jour le reste du système, ce n'est plus un problème.

D'un point de vue développeur : si vous voulez développer webkit avec llvm 3.4 et 3.3, ce n'est plus une source de problèmes.

## Mutable vs immuable

Lorsqu'on met à jour une bibliothèque, la plupart des gestionnaires de paquets la remplacent directement sur place. Toutes les nouvelles applications s'exécutent ensuite avec cette nouvelle bibliothèque sans avoir besoin d'être recompilées. Après tout, elles font toutes référence dynamiquement à `libc6.so`.

Comme les dérivations Nix sont immuables, la mise à jour d'une bibliothèque comme glibc signifie recompiler toutes les applications, car le chemin de glibc vers le Nix store a été codé en dur.

Alors comment gérer les mises à jour de sécurité ? Dans Nix, nous avons quelques astuces (toujours pures) pour résoudre ce problème, mais c'est une autre histoire.

Un autre problème est que, à moins que le logiciel ne prenne en compte un modèle purement fonctionnel, ou qu'il puisse s'y adapter, il peut être difficile d'assembler des applications au moment de l'exécution.

Prenons l'exemple de Firefox. Sur la plupart des systèmes, vous installez flash (!), et il fonctionne dans Firefox car Firefox recherche les plugins dans un chemin global.

Avec Nix, il n'y a pas de chemin global pour les plugins. Firefox doit donc connaître explicitement le chemin vers flash. La façon dont nous gérons ce problème est d'encapsuler le binaire Firefox afin de configurer l'environnement nécessaire pour qu'il trouve flash dans le Nix store. Cela va générer une nouvelle dérivation Firefox : sachez que cela prend quelques secondes, et que cela rend la composition plus difficile à l'exécution.

Avec Nix il n'y a pas de scripts d'upgrade ou de downgrade pour vos données. Cela n'a pas de sens avec cette approche, car il n'y a pas de dérivation à mettre à jour. Avec Nix, vous utiliser un autre logiciel avec sa propre pile de dépendances, mais il n'y a pas de notion formelle d'upgrade ou downgrade dans cette opération.

Si le format de données change, la migration vers ce nouveau format reste à votre charge.

## Conclusion

Nix vous permet de composer des logiciels au moment de la compilation avec un maximum de flexibilité, en veillant à ce que les builds soient aussi reproductibles que possible. De plus, grâce à cette conception de Nix, le déploiement de systèmes dans le cloud est si facile, cohérent et fiable que dans le monde Nix tous les outils d'auto-gestion et d'orchestration existants sont dépréciés par [NixOps](http://nixos.org/nixops/).

Cependant, Nix est _actuellement_ limité lorsqu'il s'agit de faire de la composition dynamique à l'exécution ou du remplacement de bibliothèques de bas niveau, en raison de la nécessité de reconstruire les dépendances.

Cela peut sembler effrayant, cependant après avoir utilisé NixOS à la fois sur un serveur et sur un ordinateur portable, j'en suis très satisfait jusqu'à présent. Certains des problèmes architecturaux nécessitent simplement de la main-d'œuvre, tandis que d'autres problèmes de conception doivent encore être résolus par la communauté.

Considérant que [Nixpkgs](https://nixos.org/nixpkgs/) ([lien github](https://github.com/NixOS/nixpkgs)) est un dépôt complètement neuf rassemblant tous les logiciels existants, avec un concept entièrement innovant, avec peu de développeurs principaux mais une augmentation régulière des contributions année après année, l'état actuel est plus que satisfaisant et dépasse largement le stade expérimental. En d'autres termes, cela vaut votre investissement.

## Next pill...

...nous allons installer Nix par-dessus votre système actuel (que je suppose être GNU/Linux, mais nous avons aussi des utilisateurs OSX) et commencer à inspecter les logiciels installés.
