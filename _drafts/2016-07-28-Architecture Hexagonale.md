---
layout: post
title:  "L'architecture Hexagonale"
date:   2016-07-28 12:00:00 +0200
---

# L'architecture hexagonale

## Context et problèmes

Le développement d'applications d'entreprise ayant une complexité métier nous confronte à un certain nombre de défis d'un point de vue architecture.

La durée de vie de celles-ci va souvent s'étaler sur plusieurs années durant lesquelles elles ne cesseront d'évoluer au fur et à mesure des nouvelles exigences et besoins.

### Le changement

Un premier de ces défis est certainement que l'architecture doit elle doit permettre le *changement*. Elle doit permettre de répondre à ces demandes d'évolution et de s'adapter à son environnement avec un time to market le plus petit possible.

#### Changement dans les moyens d’interactions

Un exemple de changements auquel une application va certainement être confrontée durant son existence est de lui permettre d'être appelée de différentes manières. Aujourd'hui, rare sont les applications business qui fonctionnent de manière autonome. S'en est fini des applications dont les cas d'utilisation sont seulement appelé via une interface web. Dans ce monde des systèmes distribués il est fréquent de devoir d'être appelé via d'autres systèmes via de multiples canaux:

- web services REST ou SOAP
- de manière asynchrone via messaging;
- batch
- etc.

L'architecture doit permettre l'ajout de nouveau moyen d'interactions sans pour autant impacter les autres ou même la logique business. Ces nouveaux moyens de communications doivent pouvoir être implémentés sans aucun impacts sur l'existant.

#### Changements technologiques

Une autre raison de changement auquel nous sommes souvent confrontés est l'évolution technologies.

Afin de ne pas réinventer la roue et augmenter notre productivité, nous utilisons généralement des frameworks. L'évolution des ces dépendances constitue également un défis. Il faut pouvoir suivre ces mises à jours tout en minimisant le risque que cela constitue. L'architecture doit permettre de suivre les nouvelles versions des dépendances utilisées sans pour autant être un big bang à chaque fois.

De plus, les frameworks et librairies répondant à nos besoins aujourd'hui ne seront peut-être plus les mêmes demain. Une architecture doit nous permettre de pouvoir répondre à ces nouveaux besoins et d'expérimenter de nouveaux outils.

Un exemple courant concerne les technologies de stockage des données. Traditionnellement les bases de données relationnelles sont les plus utilisées. Cependant, des technologies NoSQL sont dans certain cas une bien meilleure option. Cette migration, même partielle, d'une technologie à une autre représente bien souvent un risque et une charge de travail trop élevés que pour se l'autoriser.

L'architecture doit nous permettre d'expérimenter et de changer les outils que nous utilisons afin de mieux répondre à nos besoin.

#### Changements applicatifs et métiers

Evidemment, une application va également changer afin de répondre aux nouvelles exigences en terme de processus et règles métiers et applicatifs. 

### Fragilité

Cependant, l'architecture doit autoriser ces changements tout en évitant de la rendre fragile face à ceux-ci. Elle doit isoler les parties du code les plus importantes des parties qui le sont moins. 

Le cœur de l'application, composé des processus et règles métiers et applicatives, doit être protégé des évolutions dans les couches de plus bas niveau. L'architecture doit empêcher les modules haut niveau de dépendre de modules de plus bas niveau.

### Testabilité

Personnellement, je suis adepte du développement dirigés par les tests. Pourquoi c'est important qu'une application soit testable:

- Prévenir les bugs et dès lors améliorer la qualité de l'application
- être en confiance face au changement. Le changement n'est possible que si il existe des tests permettant de s'assurer que le comportement, lui, n'a pas changé.
- Si l'application est testable, cela favorise le développement dirigé par les tests. Pratique qui permet d'améliorer également la qualité du code car:
 - Le test est le premier client de l'API dès lors

Les TESTS sont LA chose la plus importantes pour s'autoriser le changement. Il faut cependant que les tests ne dépendent pas de la technologie.


## Solution

L'architecture hexagonale consiste à séparer le code d'une application en plusieurs parties.

![](/assets/posts/2016-07-28-architecture-hexagonale/hexagonal_architecture_inside.png "The code is divided in two area")

Plus on se rapproche du centre plus le code qui s'y trouve est de haut niveau.

### La règle de dépendance

La règle d'or est que **du code haut niveau ne peut sous aucune condition dépendre de code bas niveau. Visuellement, cela revient à dire que le sens des dépendances doit toujours être de l'extérieur vers l'intérieur.**

L'hexagone central est le modèle du domaine alors que l'extérieur contient les détails, les modules bas niveau, les mécanisme permettant d’appeler les cas d'utilisations via différents canaux.

### Modèle du domaine

Cette couche contient le modèle du domaine. Les règles business ne peuvent se trouver uniquement que dans cette couche. C'est le module de plus haut niveau de l'application.

### Cas d'utilisation

Cette couche contient les utilisations implémentés dans des services applicatifs. Ils implémentent la logique applicative. Ils savent quoi faire pour implémenter le scénario business. Ils coordonnent les appels aux domaine qui lui sait comment faire pour résoudre les problème du domaine.

Les cas d'utilisations vont pouvoir être appelé via n'importe quel port en fonction de l'évolution des besoins. Si un cas d'utilisation doit être appelé par un nouveau port, via JMS par exemple, il suffit d'ajouter un adaptateur qui va transformer la requête JMS en requête compréhensible par les services applicatifs.

L'API ou l'implémentation de ceux-ci ne doivent aucunement être impactés par un quelconque ajout ou changement changement des adaptateurs.

### Ports et Adaptateurs

Alistair Cockburn a également donné le nom de 'Ports and Adapters' à ce pattern d'architecture car l'application, via ses cas d'utilisation, défini des ports qui permette de l'appeler. De plus, l'application interagit avec le monde extérieur via des ports également.

C'est dans cette couche uniquement que les dépendances vers des frameworks ou librairie sont autorisées.

Par ailleurs, cette couche contient donc du code qui peut être séparé en deux catégories:

1. Les adaptateurs pour les *ports primaires* qui exposent les cas d'utilisations sur des canaux tels que le web, REST, SOAP ou messaging. Ils permettent à l'application d'être appelée par ses clients. Ces adapteurs peuvent dépendre par exemple de technologies ou frameworks comme Spring, EJB, JAX-RS, JAX-WS, JMS, etc.

2. Les adaptateurs pour les *ports secondaires* sont l'implémentation des abstractions définies par les modules haut niveau. Ces adaptateurs peuvent dépendre par exemple de technologies ou framework comme Spring Date, JPA, MongoDB, etc.

### Inversion des dépendances

La problèmatique est de permettre à du code haut niveau d'exécuter du code de plus bas niveau que lui tout en respectant la règle de dépendance: du code haut niveau ne peut jamais dépendre du code de plus bas niveau.

Les exemples de cette situations sont cependant nombreux. Dans la majorité des applications fréquents sont les cas d'utilisations qui vont devoir accéder à une source de données ou appeler un service externe (web service, e-mail, etc.).

La solution consiste à inverser le sens des dépendances. Il faut que celui-ci aille contre le sens du flux de contrôle qui va du module appelant (haut niveau) vers le module qui est appelé (bas niveau).

Afin d'illustrer ce concept prenons l'exemple d'un cas d'utilisation qui doit envoyer un e-mail à certains utilisateurs à chaque fois q'un certain évènement s'est produit. Le diagramme UML sans inversion des dépendances serait ressemblerait donc à:

![Without inversion of dependency](http://yuml.me/c7cb8842)

Le classe qui implémentate le cas d'utilisation envoi des e-mail en appelant une méthod de la classe Mailer. Mailer étant une classe de plus bas niveau que UseCaseOne cette dépendance est strictement interdite.

Afin d'inverser le sens des dépendances, il nous faut définir une abstraction *Notifier* (via une interface) dont va dépendance *UseCaseOne* et qui sera implémentée par *Mailer*.

![With inversion of control](http://yuml.me/be48254b)

Il est important de comprendre deux choses:

1. Cette abstraction exprime une dépendance de la manière la plus pratique pour le code appelant, *UseCaseOne*. C'est *UseCaseOne* qui défini l'interface de sa dépendance et non l'implémentation de *Notifier* qui défini comment le cas d'utilisation doit l'utiliser.

2. La définition de l'interface se trouve dans la même couche que le code qui en dépend, dans le cas présent dans la couche *Use cases*.

## Communication entre les couches

**L'API est toujours définie de manière la plus pratique pour la couche de plus haut niveau.**

## Conclusion

*L'architecture doit contribuer au développement d'applications agiles, robustes et testables.*
*Robuste car elle doit protéger le coeur de l'application des changements*

### Use Case Driven

La mise en place de cette architecture encourage une approche top/down. En se concentrant tout d'abord sur la compréhension et l'implémentation du comportement en abstraction totale de toute contraite technologique.

Par ailleurs, les cas d'utilisation ont une place dédiée dans l'architecture de l'application. Ce qui permet d'avoir du code qui se documente par lui même.

Contrairement a certaine implémentation de layered architecture, ou les use cases sont implémenté dans la couche services, bien souvent ils sont mixés avec du code ayant d'autres responsabilité comme l'accès à des services externe ou autre.

Les cas d'utilisations sont first component citizen. Ils ont un layer qui leur appartient. Ils ne sont plus perdu dans une couche mixant des use cases, du code fonctionnel de plus bas niveau, du code technique.

Le fait de mettre en avant ces use case permet/facile un développement TOP -> DOWN. On se concentre sur la définition du comportement business et ensuite on s'occupe des détail.

### Testable

Parfait pour faire du Test Driven Development. Etant donné que les cas d'utilisation sont de simple POJO il est très facile de commencer à définir leur comportement via des tests unitaires.

Les tests unitaires sont les premiers adapteurs de toute cas d'utilisation.

Etant donné que tout accès aux ports secondaires sont obligatoirement derrière une abstraction il est très facile de les mocker.

###  Code review

En soit cette architecture est très proche d'une architecture en couche. La seule différence réside dans la codification de certaines règles.

Les règles d'architectures sont facilement compréhensibles et communicable. J'ai pû observer que ces règles facilitaient également le code review et aidaient dans certaines discussions sur le design d'une application.

### Décision tardive


## Resources

* [Hexagonal Architecture by Alistair Cockburn](http://alistair.cockburn.us/Hexagonal+architecture)
* [The architect's clue bucket](http://www.ruthmalan.com/Journal/JournalCurrent.htm#architecture)
* [A zoom on the hexagonal/clean/onion architecture](http://tpierrain.blogspot.be/2013/08/a-zoom-on-hexagonalcleanonion.html)
* [The Clean Architecture by Uncle Bob Martin](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html)
* [Software Architecture Pattern on O'Reilly](https://www.oreilly.com/ideas/software-architecture-patterns)
*  [Agility and Software Architecture](http://www.grahamlea.com/2015/07/agility-software-architecture-simon-brown-yow-2014/)