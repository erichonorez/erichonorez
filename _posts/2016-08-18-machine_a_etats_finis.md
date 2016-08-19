---
layout: post
title:  "Machine à états finis"
date:   2016-08-18 12:00:00 +0200
comments: true
---


## La théorie

Une machine à états finis est un modèle mathématique qui permet de décrire les différents états qu'un système peut avoir ainsi que les transitions entre chacun de ceux-ci. 

Suivant ce modèle, un système ne peut être que dans un seul état à la fois et le changement de celui-ci est provoqué par un évènement.

Plus formellement, une machine à états finis se défini comme le quintuple
(`Q`, `V`, `δ`, `I`, `F`) avec:

- `Q` étant l'ensemble des états que le système peut avoir;
- `V` étant l'ensemble des évènements que le système comprend, auxquels il réagit;
- `I` étant l'état initial du système qui fait partie de Q;
- `F` est l'ensemble des états finaux du système;
- `δ` étant l'ensemble des relations de transition.

Il existe deux types de machine à états finis:

- Moore: l'état vers lequel le système va transiter à la réception d'un événement n'est fonction uniquement de l'état courant;
- Mealy: l'état vers lequel le système va transiter à la réception d'un événement est fonction de l'état courant et de l'événement reçu.

Prenons un exemple courant qui pourrait se modéliser comme une de machine à états finis: un processus Kanban. Kanban permet de visualiser et matérialiser un processus composé de plusieurs étapes que les tâches à faire vont successivement traverser. Imaginons un processus développement logiciel composés des étapes suivantes:

1. `TO DO`
2. `IN PROGRESS`
3. `WAITING_ACCEPTANCE`
4. `FOR_PRODUCTION`
5. `IN_PRODUCTION`

Ces étapes forme l'ensemble des états qu'une tâche à faire peut avoir. L'état initial est "TO DO" tandis que l'état final est "DEPLOYED". Voici les différentes relations de transition:

1. `TO DO` -> `IN PROGRESS`: lorsqu'un développeur s'assigne la tâche à faire;
2. `IN PROGRESS` -> `WAITING_ACCEPTANCE`: lorsque le code est mergé dans le repository git;
3. `WAITING_ACCEPTANCE` -> `FOR_PRODUCTION`: lorsque les utilisateurs finaux ont validé le bon fonctionnement la fonctionnalité;
4. `WAITING_ACCEPTANCE` -> `IN_PROGRESS`: lorsque les utilisateurs finaux n'ont pas validés le bon fonctionnement de la fonctionnalité et que la tâche à faire est automatiquement réassignée au développeur;
5. `FOR_PRODUCTION` -> `IN_PRODUCTION`: lorsque la fonctionnalité est déployée en production.

Ces relations de transition sont toujours une fonction de type: (Etat courrant x Evénement) -> nouvel état.

![Final State Machine](/assets/posts/2016-08-18-machine-a-etats-finis/kanban_fsm.png "Processus de développement")

## Pourquoi s'intéresser aux machine à états finis?

Dans le développment d'applications il est fréquent de se retrouver à gérer les différents états possibles des entités de notre domaine. De mon exéprience la gestion de ceux-ci peut devenir de plus en plus compliquée au fur et à mesure de l'ajout de nouveaux cas d'utilisation. Imaginez une application qui permet de gérer des Kanbans et que chacun matérialise un processus différent. Chaque processus aura un ensemble d'états, d'événements, d'actions et de transitions différents. De plus, lorsque le code qui change les états du système est dissiminé dans plusieurs services, il est vite difficile de savoir en un coup d'oeil ce qu'il se passe dans tel ou tel processus quand tel ou tel évènement se produit.

Une première raison est donc rassembler la responsabilité de changer les états derrière une unique API.

Ensuite, la modélisation d'une machine à état finis va également permettre de rendre des concepts implicites, explicites. Généralement, les différents états possibles d'un système apparaissent clairement dans le code, sous la forme d'un Enum par exemple. Cependant, les évènements qui provoquent les transitions entre les états sont rarement (de mon expérience) explicites dans le code. Bien souvent se sont des services qui lors de l'éxécution de leurs méthodes vont changer l'état de tel ou tel entité.

En définissant les différents évènements d'un système à états finis, nous explicitons clairement qu'un changement d'état est provoqué par un évènement précis. Le résultat est un code plus compréhensible et qui se documente lui-même.

De plus, le fait de se concentrer sur la modélisation d'une machine à état finis et de rendre explicite ses évènements en leur donnant des noms, aide à la réflexion et permet de découvrir de nouveaux concepts du domaine, de nouveaux évènements et états. En résumé, cela contribue à obtenir une perceptions plus profonde du domaine.

Enfin, le fait de donner des noms aux évènements qui produisent des transitions d'état favorise une meilleure compréhension lors des discussions avec les experts métier. Parler d'un évènement qui produit un changement d'état est plus compréhensible que de "quand le code appelle cette méthode, le statut change". 
Idéalement, les discutions avec les experts vont permettre de choisir judicieusement les noms des états et évènements afin d'avoir un langage omniprésent: dans le code et dans les discussions.

## Dans la pratique

Une machine à états finis reviens à implémenter la fonction suivante:

`State(S) x Event(E) -> Actions (A), State(S')`

J'ai créé un projet qui m'a permit d'expérimenter les concepts et d'avoir une implémentation minimaliste. Le code source des exemples se trouvent [ici](https://github.com/erichonorez/finite-state-machine/tree/master/fsm).

La définition d'une FSM se fait en définissant les différents états et évènement en se servant (dans mes exemples) d'enums:

```java

public enum ProcessStage implements State {
    TODO(false),
    IN_PROGRESS(false),
    WAITING_ACCEPTANCE(false),
    FOR_PRODUCTION(false),
    IN_PRODUCTION(true);

    private final boolean isFinal;

    ProcessStage(boolean isFinal) {
        this.isFinal = isFinal;
    }

    @Override
    public boolean isFinal() {
        return this.isFinal;
    }
}

```

```java

public enum ProcessEvent implements Event {
    WORK_ITEM_ASSIGNED,
    CODE_COMMITED,
    ACCEPTED,
    REFUSED,
    DEPLOYED
}

```

Voici la définition des transitions entre les différents états:

```java

	startWith(ProcessStage.TODO).transit(

        on(ProcessEvent.WORK_ITEM_ASSIGNED)
            .to(ProcessStage.IN_PROGRESS).transit(

                on(ProcessEvent.CODE_COMMITED)
                    .to(ProcessStage.WAITING_ACCEPTANCE).transit(

                        on(ProcessEvent.ACCEPTED)
                            .to(ProcessStage.FOR_PRODUCTION).transit(

                                on(ProcessEvent.DEPLOYED)
                                    .to(ProcessStage.IN_PRODUCTION)
                ),

                on(ProcessEvent.REFUSED)
                    .to(ProcessStage.IN_PROGRESS)
            )
        )
    );
            
```

Afin de faire transiter la FSM d'un état à l'autre nous intéragissons de la manière suivante:

```java

@Test
    public void WhenProcessReceiveAKnownEvent_ItShouldTransitToAnotherState() {
        Process process = new Process();
        ProcessContext initialContext = new ProcessContext(ProcessStage.TODO);

        ProcessContext contextInTodoState = process.trigger(ProcessEvent.WORK_ITEM_ASSIGNED, initialContext);
        Assert.assertEquals(ProcessStage.IN_PROGRESS, contextInTodoState.currentState());

        ProcessContext contextInWaitingAcceptanceState = process.trigger(ProcessEvent.CODE_COMMITED, contextInTodoState);
        Assert.assertEquals(ProcessStage.WAITING_ACCEPTANCE, contextInWaitingAcceptanceState.currentState());

        ProcessContext contextInForProductionState = process.trigger(ProcessEvent.ACCEPTED, contextInWaitingAcceptanceState);
        Assert.assertEquals(ProcessStage.FOR_PRODUCTION, contextInForProductionState.currentState());

        ProcessContext contextInProductiontate = process.trigger(ProcessEvent.DEPLOYED, contextInForProductionState);
        Assert.assertEquals(ProcessStage.IN_PRODUCTION, contextInProductiontate.currentState());
    }
    
```

Ces examples permettent de constater que l'utilisation d'une machine à états finis permet d'avoir un code relativement simple et explicite.

Par ailleurs dans les exemple la classe `FSMGraphvizGenerator.java` permet de facilement exporter la définition d'une FSM sous la forme d'un graphviz:

```java
FSMGraphvizGenerator.generate(new Process());
```

Les exemples ci-dessus sont relativement simples. Le but était d'illustrer ce qu'est une machine à états finis et comment elles peuvent contribuer à avoir un code clair et plus compréhensible. Il est évident que dans la pratique les cas sont souvent plus complexes...


## Prochaine étape

En faisant quelques recherches sur google j'ai pû trouver des implémentations de machines à états finis qui me paraissent intéressantes et qui pourraient être utilisée pour des applications d'entreprise:

* [Squirrel](https://github.com/hekailiang/squirrel)
* [Spring statemachine](https://projects.spring.io/spring-statemachine/)
* [Akka FSM](http://doc.akka.io/docs/akka/current/scala/fsm.html)

La suite, peut-être dans un prochain article. 

## Resources

* [Les slides d'un cour de l'université de Nice](http://www.i3s.unice.fr/~fedou/wwwUnice/CoursOFI_files/OFI8.pdf)
* [La documentation de Akka](http://doc.akka.io/docs/akka/current/scala/fsm.html)
* [Event Storming](http://ziobrando.blogspot.be/2013/11/introducing-event-storming.html)
* [Video d'explication sur YouTube (en anglais)](https://www.youtube.com/watch?v=hJIST1cEf6A)
* [Automates d'Etats finis sur YouTube (en français)](https://www.youtube.com/watch?v=tzyCzbWjNY0)