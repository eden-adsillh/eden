# Projet de Programmation réseau - Rapport
Dans le cadre du cours de réseau de la licence pro ADSILLH, il est demandé de
transformer le code d'un jeu en Python pour le mettre en réseau par groupe de
deux au moyen de nos connaissances.

Dans ce rapport, le terme *serveur* fait référence à l'hôte de la partie et le
terme *client* fait référence à n'importe quel autre joueur.

Ce rapport est rédigé par Yorick Barbanneau et Luc Lauriou.

## Fonctionnalités apportées
Notre solution embarque comme nouvelles fonctionnalités, par rapport au code
fourni, le multijoueur en réseau à deux, ou à trois avec la possibilité
d'incarner le serpent et l'obfuscation des messages pour éviter la triche.

## Déroulement
Dans un premier temps, on a choisit de faire une solution de communication
réseau chacun de son coté puis de prendre le meilleur des deux solutions.
Ensuite, on a travaillé chacun de son coté sur un module complémentaire.

Comme point de départ, la fonction `select` de Python placée dans la boucle
infinie du jeu semblait intéressante. Cependant, l'appel de
`pygame.event.wait` étant bloquant dans cette boucle, il a été décidé de créer
un thread parallèle contenant ce select et écoutant les sockets pour éviter
d'éventuels problèmes à venir, en prenant pour exemple la façon dont
fonctionnent d'autres outils du monde du jeu vidéo : une boucle, chargée de
l'affichage, effectue le moins d'actualisations possible, avec comme limite
maximum, de préférence, le taux de rafraichissement du moniteur qui affiche le
jeu, et une boucle, chargée d'effectuer d'autres calculs, actualisée le plus
souvent possible.

Ensuite, on a effectué la mise en place du module pour jouer avec le serpent.
Bien que, jusqu'alors, la présence d'un `select` dans le thread ne s'avérait pas
nécessaire car, étant 2 joueurs, chacun n'avait qu'un socket à surveiller (celui
de l'opposant), elle devient maintenant indispensable. Pour éviter d'avoir un
serveur central, on a d'abord décidé que les clients devaient, par défaut,
supposer qu'ils devaient accepter une éventuelle nouvelle connexion, bien que
seul un dernier client puisse se connecter et qu'il n'est pas nécessaire pour
les deux d'attendre une connexion (car seul l'un d'entre eux va se connecter à
l'autre et qu'il n'y aura pas plus de trois joueurs).

## Difficultés rencontrées
Des difficultés ont été rencontrées lors de la mise en place de cette solution.
Tout d'abord, il m'a fallu du temps pour comprendre que la fonction `accept`
de la classe `socket` s'est retrouvée bloquante, même dans le deuxième thread,
tant que le `select` n'était pas mis en place.\
Ensuite, je m'attendais à ce que la fonction `connect` de la classe `socket`
retourne le socket sur lequel on tente de se connecter, hors il n'en est rien.
La solution a été d'utiliser `socket.create_connection` à la place.

La principale difficulté de la mise en place du serpent a été de se poser la
question de la gestion des sockets avec un troisième joueur sans serveur
central. Les joueurs ne sont pas identifiés par leurs adversaires par les
messages qu'ils envoient mais par leurs sockets. Partageant le même code, la
précédente solution qui était de stocker la socket locale sous la variable `so`
et une socket distante sous la variable `opponent_so` s'avère ardue, même en
ajoutant une variable `snake_so` : comment fait le serpent pour savoir quelle
socket représente la femme et laquelle représente l'homme ? Il a donc été décidé
de conserver la variable `so` pour la socket locale, et d'avoir trois socket,
`woman_so`, `man_so` et `snake_so` pour identifier facilement à quel joueur
appartient quelle socket (avec cette solution, une des trois variable reste
cependant vide).

Un bug relatif au fait que la gestion de l'affichage et des évènements claviers
soit dans une boucle parallèle à la gestion des sockets est apparu. Si un joueur
joue un mouvement alors que tout le monde n'a pas actualisé l'affichage, il
n'aura plus le même affichage que les autres et aura, grossièrement, "un coup
d'avance". Deux solutions ont été envisagées : stocker les mouvements envoyés
par les sockets et les effectuer récursivement ou empêcher quiconque de jouer
tant qu'il n'a pas reçu un message "READY" de la part de tous les autres
joueurs.

### Mode de communication "serverless"

Permettre un mode de jeu réseau pour plus de deux joueur sans serveur central
fut difficile. Le fait d'avoir des joueurs définis (Adam, Eve et le Serpent) en
rajoute une couche. Il nous a fallu définir un protocole d'échange
d'informations entre pairs précis :

![fonctionnement de l'initialisation d'un connexion](imgs/net_init.svg)

 1. Le joueur 1 se connecte au joueur 2 qui accepte la connexion.
 2. Le joueur 2 envoi la liste des joueurs auxquels il est connecté par des
    messages `/peer <adresse:port>`. Le joueur 1 ne traitera les messages
    `/peer` venant seulement du joueur 2
 3. Le joueur 2 envoi le personnage qu'il joue par un message `/player <name>`

Le joueur 1 ignorera toutes les requêtes `/peer` sauf celles venant du joueur 2
afin d'éviter les problèmes de connexion infinie ( 1 -> 2 -> 3 -> 2 ->3 etc.) 

### Obfuscation des mouvements et triche

L'obfuscation est obtenue en appliquant la fonction `crypt.crypt()`. La mise en
œuvre de cette fonctionnalité est bien plus subtile qu'il n'y parraît.

#### Utiliser un sel "fort"

Afin de créer un sel permettant un chiffrement fort du message et éviter ainsi
une attaque par *bruteforce*, nous avons utilisé la fonction `crypt.mksalt()` de
la façon suivante : 

```python
key = crypt.mksalt(crypt.METHOD_SHA256)
```

#### Envoyer seulement le message hashé

la fonction `crypt.crypt()` retourne une chaîne de caractères reprenant le sel
puis le message hashé. En envoyant l'ensemble de cette chaîne, les adversaires
seraient en mesure de déchiffrer le message facilement. Il faut donc envoyer
seulement la partie hashée du message :

```python
msg = crypt.crypt( current_user['move'], current_user['key']).split('$')[3]
create_message(0,msg)
```
Voir l'annexe 1 pour une PoC

#### Changer la clé à chaque tour

Comme la vérification des messages nécessite l'envoi du sel aux adversaires, il
pourraient essayer de déchiffrer les message suivants avec la clé obtenue. Il
est donc nécessaire de changer la clé à chaque tour.

## Sources

## Annexes

### Annexe 1

Petit programme python permettant de déchiffrer un message contenant l'ensemble
de la chaîne résultant de  la fonction `crypt.crypy()`

```python
#!/bin/python
import sys
import crypt

mhash = input("Enter hash to analyse : ")
key = mhash.split('$')[2]
method = mhash.split('$')[1]
for i in ['LEFT', 'DOWN', 'UP', 'RIGHT']:
    test = crypt.crypt(i, '$' + method + '$' + key)
    if test == mhash:
        print('Played: %s' % (i))
        break
```

### Annexe 2

Petit programme en python permettant de déchiffer les messages envoyés par un
jouer s'il ne change pas de sel cryptographique.

```python
#!/bin/python
import sys
import crypt

mhash = input("Enter hash to analyse : ")
key = input("Enter the key : ")

for i in ['LEFT', 'DOWN', 'UP', 'RIGHT']:
    test = crypt.crypt(i, key).split('$')[3]
    if test == mhash:
        print('Played: %s' % (i))
        break
```
