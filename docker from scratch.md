# Exécuter une image docker, mais sans docker

## Pourquoi faire?

Vous voulez dire à part avoir du bon gros fun, comme on dirait au Québec?

Eh bien, car c'est, je pense, un bon moyen de comprendre ce que fait docker, et comment il le fait. Nous permettant entre autre de se représenter un peu plus les mots prononcés par certains barbus autour de la machine à café ou dans un meetup, comme:

* Layers, overlayfs, unionfs
* chroot
* namespace
* cgroup
* linux capabilities

<img src="https://media.giphy.com/media/XZn3XNxcxpv9ftdgdm/giphy.gif"></img>


Une fois qu'on aura balayé ces concepts, on abordera un peu les aspects sécurité, car comme a dit un grand philosophe: 
> "Un grand pouvoir implique de grandes responsabilités" - Benjamin Parker

## A vos shells <span style="font-size:x-small">(Linux, car je n'ai pas de Mac, et que je lance Windows que pour jouer)</span>!


## Récupérer une image docker au format .tar

Pour cet article, nous allons partir d'une image nginx.

### Création d'un système de fichier pour notre container (version simpliste)

Je sais que je vous ai promis de faire du docker sans docker, mais il faut à minima avoir une image à un format exploitable:

<code>
	docker pull nginx:alpine
	
	docker save nginx:alpine > nginx.tar

	mkdir nginx-kind-of-container
	
	cd nginx-kind-of-container
	
    # Création de l'arborescence du conteneur à exécuter, détails ci-dessous
	tar --to-stdout -xvf ../nginx.tar manifest.json | jq .[0].Layers[] -r | xargs -n1 -I{} tar --overwrite -xvf  ../nginx.tar {} --to-command="tar xvf -" 

</code>

### Les explications

___
On commence donc par récupérer l'image en local:

<code>docker pull nginx</code>

___

Cependant cela ne nous fournit pas un format utilisable en dehors du démon docker, pour cela, nous utilisons la commande ci-dessous, qui va exporter son contenu dans une archive .tar:

<code>docker save nginx > nginx.tar</code>

Comme vous le savez surement si vous avez déjà ouvert un Dockerfile, une image docker hérite d'une image, qui elle-même hérite d'une autre image, et ainsi de suite jusqu'à l'image <code>scratch</code> qui correspond à un système de fichier vide.

*Note: Ceci est une description simplifiée, car une image peut être composée elle-même de plusieurs couches*

Ce fonctionnement signifie:

* Deux images qui ont le même parents signifie que l'image parente n'est récupérée qu'une seule fois localement
* Economiser de l'espace sur son disque dur en utilisant des systèmes de fichiers en couche, comme Layerfs, unionfs ou autre (plus loin dans l'article)
* Permet la réutilisation d'images de bases plutôt que de réinventer la roue (si on a besoin de servir des pages html, il suffit de faire FROM nginx au début de notre Dockerfile, et ajouter nos fichiers)


Voici le contenu de l'archive nginx.tar:

```
.
├── 3540a11f07a07eb51b115ed3a03ff2a1c5a9fe668d69d8e2b6ca585d1d35bb19 << (A)
│   ├── json
│   ├── layer.tar
│   └── VERSION
│
├── ... << (A)
│   ├── json
│   ├── layer.tar
│   └── VERSION
│
├── ed21b7a8aee9cc677df6d7f38a641fa0e3c05f65592c592c9f28c42b3dd89291.json << (B)
├── manifest.json (C)
└── repositories (D)

```



(A) Correspond à une couche du conteneur nginx que nous créons. Les fichiers de cette couche sont présent dans l'archive layer.tar

(B) Contient les variables d'environnement à injecter lors de la création du conteneur (par soucis de simplification, on ne s'en occupera pas dans cet article)

`(`C`)` Description de l'ordre des couches et le nom de l'image docker, la partie qui nous intéresse ici est "Layers", qui définit l'ordre dans lequel appliquer les couches <b>(A)</b> pour la création de notre conteneur:

```
[
  {
    "Config": "ed21b7a8aee9cc677df6d7f38a641fa0e3c05f65592c592c9f28c42b3dd89291.json",
    "RepoTags": [
      "nginx:latest"
    ],
    "Layers": [
      "90b1753aa437ddb654064610bc4884bf2328edac058afed97a4e5b00c32db945/layer.tar",
      "3540a11f07a07eb51b115ed3a03ff2a1c5a9fe668d69d8e2b6ca585d1d35bb19/layer.tar",
      "a758b9aae4794392c03d8cd8541e22058b072fab7ff50abcf1e6570675dea24e/layer.tar"
    ]
  }
]

```

(D) Nom de l'image courante ainsi que son tag

___

Et maintenant la commande un peu moins... lisible: 

<code>tar --to-stdout -xvf ../nginx.tar manifest.json | jq .[0].Layers[] -r | xargs -n1 -I{} tar --overwrite -xvf  ../nginx.tar {} --to-command="tar xvf -"</code>

* <code>tar --to-stdout -xvf ../nginx.tar manifest.json</code>: on commence par lire le contenu de manifest.json
* <code>jq .[0].Layers[] -r</code>: on récupère l'ordre dans lequel appliquer les couches constituant le conteneur final
* <code>xargs -n1 -I{} tar --overwrite -xvf  ../nginx.tar {} --to-command="tar xvf -"</code> : extraire les différents layers dans le même dossier, en écrasant les fichiers existant (exemple: redéfinition d'un fichier de configuration d'une image parente)

___

## Première exécution de notre conteneur

### Objectif

Nous avons maintenant dans le dossier <code>nginx-kind-of-container</code> une arborescence similaire à ce que nous aurions dans un conteneur exécuté. Cependant, il nous manque une étape importante: lancer l'exécutable <code>dans le bon context</code>

### Première tentative

<code>
	[mpayetippon@clyde-fedora nginx-kind-of-container]$ ./usr/sbin/nginx -v
	
	./usr/sbin/nginx: error while loading shared libraries: libpcre.so.3: cannot open shared object file: No such file or directory


	[mpayetippon@clyde-fedora nginx-kind-of-container]$ ls ./lib/x86_64-linux-gnu/libpcre.so.3

./lib/x86_64-linux-gnu/libpcre.so.3
</code>

Malgré la présence du binaire <code>nginx</code> et de la librairie <code>libpcre.so.3</code>, nginx ne s'exécute pas, car nginx-kind-of-container/lib/x86_64-linux-gnu ne fait pas partie des dossiers dans lesquels chercher des librairies

### Utilisation de chroot

```
[mpayetippon@clyde-fedora nginx-kind-of-container]$ pwd
/home/mpayetippon/Articles/scripts/sandbox/nginx-kind-of-container

[mpayetippon@clyde-fedora nginx-kind-of-container]$ ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

[mpayetippon@clyde-fedora nginx-kind-of-container]$ ls /
bin  boot  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  snap  srv  sys  @System.solv  tmp  usr  var

[mpayetippon@clyde-fedora nginx-kind-of-container]$ sudo chroot . /bin/sh

root@clyde-fedora:/# pwd
/

/# ls 
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var

/# ls /
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var

```

La commande permet de créer un contexte dans la racine du système (/) est déplacé.

On passe donc:

* de <code>./usr/sbin/nginx</code> à <code>/usr/sbin/nginx</code> 
* de <code>./lib/x86_64-linux-gnu/libpcre.so.3</code> à <code>/lib/x86_64-linux-gnu/libpcre.so.3</code>


Maintenant, nous avons tout ce qu'il faut pour que nginx trouve ses librairies:

```
[CHROOT] /# /usr/sbin/nginx -v
nginx version: nginx/1.17.9

```

Et c'est un succès! 

### Dossiers et fichiers spéciaux

Maintenant, exécutons nginx, voir ce qui se passe:

```
[CHROOT] /# nginx
nginx: [emerg] open("/dev/null") failed (2: No such file or directory)

```

Certains dossiers et fichiers sous Linux ont des comportement particuliers, comme par exemple /dev/null. Dans un autre terminal que celui dans lequel nous avons exécuté chroot, on peut faire les commandes suivantes pour renseigner ces dossiers

```
 [mpayetippon@clyde-fedora nginx-kind-of-container]$ sudo mount -o bind /dev/ ./dev/ #cette commande va lier /dev et ./dev

 [mpayetippon@clyde-fedora nginx-kind-of-container]$ sudo mount -t proc none ./proc #./proc contiendra maintenant les processus en cours d'exécution (pid et contexte d'exécution)

```


Maintenant:

```
[CHROOT] / # pgrep nginx
[CHROOT] / # nginx
[CHROOT] / # pgrep nginx
40360
40361
40362
40363
40364
[CHROOT] / # killall nginx
[CHROOT] / # pgrep nginx

```


## Les namespaces

### Nginx est-il vraiment isolé ?

<img src="https://media.giphy.com/media/J398B8HoBvU7Ut6VIU/giphy.gif"/>

On se sent isolé, mais faisons un petit test, par exemple en récupérant le process id de firefox, lancé sur ma machine:

```
[CHROOT] / # pgrep firefox
6585
6667
6715
6730


[CHROOT] / # ls -d /proc/*[0-9]*
/proc/1              /proc/1450           /proc/1653           /proc/2223           /proc/2465           /proc/3094           /proc/535            /proc/8721
...

```


### Namespace PID

#### Création d'un namespace PID

Ce conteneur en sait un peu trop... nous allons créer un namespace de type PID pour restreindre cette visibilité. 

Pour cela nous allons utiliser la commande unshare, qui permet de créer un processus en lui associant un ou des types de namespaces. Nous créons ici un namespace pour isoler les processus 

```
[mpayetippon@clyde-fedora nginx-kind-of-container]$ ls /proc/*[0-9]* -d
/proc/1      /proc/1075   /proc/1223  /proc/1531  /proc/160   /proc/170   /proc/1862   /proc/22349  /proc/2311  /proc/2445  /proc/2528   /proc/2765   /proc/35945  /proc/4124  /proc/6372  /proc/6848  /proc/8724  /proc/959
...


[CHROOT] / # pgrep firefox
6585
6667
6715
6730

[mpayetippon@clyde-fedora nginx-kind-of-container]$ touch ../pid_namespace
[mpayetippon@clyde-fedora nginx-kind-of-container]$ sudo unshare --pid=$(pwd)/../pid_namespace --fork --mount-proc=$PWD/proc chroot . /bin/sh

```


Nous sommes maintenant dans un contexte chroot, mais avec un nouveau namespace pour les PIDs, non partagé avec le reste de la machine.


```
/ # export PS1="[CHROOT] $PS1"

[CHROOT] / # ls /proc/*[0-9]* -d
/proc/1  /proc/brcm_monitor0


[CHROOT] / # pgrep firefox
# ne retourne rien

[CHROOT] / # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    4 root      0:00 ps aux
    
/ # ls /proc/*[0-9]* -d
/proc/1              /proc/brcm_monitor0

```


#### Rejoindre le namespace que l'on vient de créer

** pas forcément utile **


On peut retrouver notre namespace

```
...
4026533576 pid       1 45140 root        /bin/sh

```

