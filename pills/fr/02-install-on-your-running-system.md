# L'installer sur votre système

Bienvenue sur la deuxième Nix pill. Dans la [première](01-why-you-should-give-it-a-try.md) pill, nous avons brièvement décrit Nix.


Nous allons maintenant installer Nix sur notre système et comprendre ce que cela a changé. **Si vous utilisez NixOS, Nix est déjà installé ; vous pouvez passer à la [prochaine](03-enter-environment.md) pill.**

Pour les instructions d'installation, veuillez vous référer au manuel pour [installer Nix](https://nix.dev/manual/nix/stable/installation/installing-binary).

## Installation

Ces articles ne constituent pas un tutoriel sur _l'utilisation_ de Nix. Au lieu de cela nous allons parcourir le système Nix pour en comprendre les fondamentaux.

Première chose à retenir : les dérivations dans le _Nix store_ se réfèrent à d'autres dérivations qui se trouvent également dans le _Nix store_. Elles n'utilisent pas `libc` du système ou d'ailleurs. C'est un _store_ autonome avec tous les logiciels néssessaires pour construire n'importe quel paquet.

<div class="info">

Remarque : dans le cas d'une installation multi-utilisateur, comme celle de NixOS, le store appartient à root et plusieurs utilisateurs peuvent installer et construire des logiciels via un démon Nix. Vous pouvez en lire davantage sur les [installations multi-utilisateurs ici](https://nix.dev/manual/nix/stable/installation/installing-binary#multi-user-installation).

</div>

## Les prémices du Nix store

Commençons par regarder la sortie de la commande d'installation :

```
copying Nix to /nix/store..........................
```

C'est le `/nix/store` dont nous parlions dans le premier article. Nous y copions les logiciels nécessaires pour démarrer un système Nix. Vous pouvez y voir bash, coreutils, la chaîne d'outils du compilateur C, les bibliothèques perl, sqlite et Nix lui-même avec ses propres outils et libnix.

Vous avez peut-être remarqué que `/nix/store` peut contenir non seulement des répertoires, mais aussi des fichiers, toujours sous la forme «hash-name».

## La base de données Nix

Juste après la copie du store, le processus d'installation initialise une base de données :

```
initialising Nix database...
```

Nix possède également une base de données. Elle est enregistrée sous `/nix/var/nix/db`. C'est une base de données sqlite qui garde la trace des dépendances entre les dérivations.

Le schéma est très simple : il y a une table de chemins valides qui associe un integer auto-incrémenté à un chemin dans le store.

Il y a ensuite une relation de dépendance entre le chemin et les chemins dont il dépend.

Vous pouvez consulter la base de données en installant sqlite (`nix-env -iA sqlite -f '<nixpkgs>'`) puis en exécutant `sqlite3 /nix/var/nix/db/db.sqlite`.

<div class="info">

Remarque : Si c'est la première fois que vous utilisez Nix après l'installation initiale, n'oubliez pas de quitter et redémarrer vos terminaux afin de mettre à jour votre environnement shell.

</div>

<div class="warning">

Important : ne modifiez jamais `/nix/store` manuellement. Si vous le faites, il ne sera plus synchronisé avec la base de données sqlite, sauf si vous savez _réellement_ ce que vous faites.

</div>

## Le premier _profile_

Avec l'installation, nous rencontrons ensuite le concept de [profile](https://nix.dev/manual/nix/stable/package-management/profiles) :

<pre><code class="hljs">creating /home/nix/.nix-profile
installing 'nix-2.1.3'
building path(s) `/nix/store/a7p1w3z2h8pl00ywvw6icr3g5l9vm5r7-<b>user-environment</b>'
created 7 symlinks in user environment
</code></pre>

Un _profile_ dans Nix est un concept général et pratique pour réaliser des rollbacks. Les _profiles_ sont utilisés pour assembler des composants répartis sur de multiples chemins sous un nouveau chemin unifié. De plus les _profiles_ sont constitués de plusieurs "générations" : ils sont versionnés. Chaque fois que vous modifiez un _profile_ une nouvelle génération est créée.

Les générations peuvent être modifiées et restaurées de manière atomique, ce qui les rend pratiques pour gérer les modifications apportées à votre système.

Regardons notre _profile_ de plus près :

<pre><code class="hljs">$ ls -l ~/.nix-profile/
bin -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-<b>nix-2.1.3</b>/bin
[...]
manifest.nix -> /nix/store/q8b5238akq07lj9gfb3qb5ycq4dxxiwm-<b>env-manifest.nix</b>
[...]
share -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-<b>nix-2.1.3</b>/share
</code></pre>

La dérivation `nix-2.1.3` dans le Nix store correspond à Nix lui-même, avec ses binaires et bibliothèques. Le processus "d'installation" de la dérivation reproduit essentiellement dans le _profile_ la hiérarchie de la dérivation `nix-2.1.3` du store à l'aide de liens symboliques.

Le contenu de ce _profile_ est spécial, car un seul programme y a été installé. Par conséquent le répertoire `bin` pointe vers le seul programme qui a été installé (Nix lui-même).

Mais ce n'est que le contenu de la dernière génération de notre _profile_. En fait, `~/.nix-profile` est lui-même un lien symbolique vers `/nix/var/nix/profiles/default`.

Il s'agit à son tour, d'un lien symbolique vers `default-1-link` dans le même répertoire. Cela signifie donc que c'est la première génération du _profile_ `default`.

Enfin, `default-1-link` est un lien symbolique vers la dérivation "user-environment" du Nix store que vous avez vue s'afficher lors du processus d'installation.

Nous aborderons plus en détail `manifest.nix` dans le prochain article.

## Expressions Nixpkgs

Plus d'informations en provenance du programme d'installation :

```
downloading Nix expressions from `http://releases.nixos.org/nixpkgs/nixpkgs-14.10pre46060.a1a2851/nixexprs.tar.xz'...
unpacking channels...
created 2 symlinks in user environment
modifying /home/nix/.profile...
```

Les expressions Nix sont écrites en [langage Nix](https://nix.dev/tutorials/nix-language) et servent à décrire les paquets et comment les construire. [Nixpkgs](https://nixos.org/nixpkgs/) est le dépôt qui contient toutes les expressions : <https://github.com/NixOS/nixpkgs>.

Le programme d'installation a téléchargé les descriptions des paquets depuis le commit `a1a2851`.

Le second _profile_ que nous découvrons est le _profile_ des _channels_. `~/.nix-defexpr/channels` pointe vers `/nix/var/nix/profiles/per-user/nix/channels` qui pointe vers `channels-1-link` qui pointe vers un répertoire du Nix store contenant les expressions Nix téléchargées.

Les _channels_ sont un ensemble de paquets et d'expressions disponibles au téléchargement. C'est similaire aux dépôts Debian stable et instable, il y a un _channel_ stable et instable. Avec cette installation, nous suivons `nixpkgs-unstable`.

Ne vous inquiétez pas pour les expressions Nix pour l'instant, nous y reviendrons plus tard.

Enfin, pour votre commodité, le programme d'installation a modifié `~/.profile` afin d'entrer automatiquement dans l'environnement Nix. Ce que fait réellement `~/.nix-profile/etc/profile.d/nix.sh`, c'est simplement ajouter `~/.nix-profile/bin` à `PATH` et `~/.nix-defexpr/channels/nixpkgs` à `NIX_PATH`. Nous aborderons `NIX_PATH` plus tard.

Lisez le fichier `~/.nix-profile/etc/profile.d/nix.sh` il est court.

## FAQ : Puis-je renommer /nix en autre chose ?

C'est possible, mais il y a une bonne raison de continuer à utiliser `/nix`. Toutes les dérivations dépendent d'autres dérivations en utilisant des chemins absolus. Nous avons vu dans le premier article que bash faisait référence à un `glibc` avec un chemin absolu spécifique dans `/nix/store`.

Vous pouvez le vérifier par vous-même (ne vous inquiétez pas si vous voyez plusieurs dérivations pour bash) :

```console
$ ldd /nix/store/*bash*/bin/bash
[...]
```

Conserver le store dans `/nix` signifie que nous pouvons récupérer le cache binaire de nixos.org (tout comme vous récupérez des paquets depuis les miroirs debian) sinon :

- `glibc` serait installé dans `/foo/store`

- Par conséquent bash devrait pointer vers `glibc` dans `/foo/store`, au lieu de `/nix/store`

- Donc le cache binaire ne peut plus nous aider, car nous aurions besoin d'un bash _différent_, et nous devrions tout recompiler nous-mêmes.

Après tout `/nix` est un emplacement judicieux pour le store.

## Conclusion

Nous avons installé Nix sur notre système, entièrement isolé et appartenant à l'utilisateur `nix`, car nous sommes encore en train de nous familiariser avec ce nouveau système.

Noux avons appris quelques nouveaux concepts comme les _profiles_ et les _channels_. Avec les _profiles_, nous sommes capables de gérer plusieurs générations d'une composition de paquets, tandis qu'avec les _channels_, nous sommes capables de télécharger des binaires depuis `nixos.org`.

Tout a été installé sous `/nix`, avec quelques liens symboliques dans le répertoire personnel de l'utilisateur Nix. C'est parce que chaque utilisateur est capable d'installer et d'utiliser des logiciels dans son propre environnement.

J'espère n'avoir rien laissé dans l'ombre qui pourrait vous faire croire qu'il y a une sorte de magie à l'œuvre en arrière-plan. Il s'agit simplement de placer des composants dans le _store_ et de les relier avec des liens symboliques.

## Next pill...

...nous allons entrer dans l'environnement Nix et apprendrons à interagir avec le _store_.
