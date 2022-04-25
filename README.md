# ReactiveX programming for dummies

## But de ce repository

J'ai écrit ce repository dans le but de donner le maximum de connaissances autours de la programmation ReactiveX / RxJava pour pouvoir commencer à manipuler toutes les bibliothèques ReactiveX. Ce document n'est pas exhaustif mais tente de tracer les grandes lignes, l'expérience fera le reste.

## La théorie avant la pratique

Avant d'aller dans la pratique, il est très important de comprendre le système dans lequel nous développons. En règle générale, que cela soit côté front ou côté back-end, nous développons une application qui va s'exécuter sur un **système d'exploitation**. Elle va donc prendre la forme d'un [processus](https://fr.wikipedia.org/wiki/Processus_(informatique)) au sein même de ce système et avoir accès à des ressources hardwares pour son fonctionnement.

La majorité du temps, le processus de notre application ne contiendra qu'un seul [thread](https://fr.wikipedia.org/wiki/Thread_(informatique)), que l'on va nommer le thread principal *main thread* ou bien *UI thread* selon les systèmes. Nous allons, tout au court de notre explication, nous centrer autours de RxJava / Android pour contextualiser. Sur Android, le *main thread* est en charge de :
- Affichage et rafraîchissement de la vue
- L'écoute des évènements de l'utilisateur / système
- L'exécution des animations
- D'autres choses que j'ai pu oublier !

### En synthèse

Le *main thread* joue un rôle majeur dans l'exécution de l'application car si jamais il doit réaliser une opération longue ou se retrouver bloqué avec un `while(true)` par exemple, l'application ne répondrait plus aux évènements utilisateurs.

> Sur Android, si le thread graphique met plus de deux secondes à réaliser sa routine de rafraîchissement de la vue, l'OS fait apparaître une pop-up ANR (Android Not Responding) à l'utilisateur qui peut donc fermer l'application en pensant qu'elle a crash alors que cette dernière réalise simplement un appel distant ou un algorithme complèxe.

En lisant ces deux dernières lignes, on se rend donc compte qu'il est primordial d'offrir un maximum de temps et de ressources au rafraîchissement de la vue afin d'assurer la meilleure expérience utilisateur possible. 

> C'est très facile de dire ça, mais comment fait-on pour réaliser des appels distants, des requêtes en base de données, des algorithmes longs et complexes si je ne peux pas les exécuter sur le thread principal ?

## Dive in the threads

Pour pouvoir réaliser des opérations longues ou coûteuses en ressource, nous pouvons nous appuyer sur les threads. Pour rappel, un processus peut contenir un à N threads, et chacun des threads possède sa propre [pile d'exécution](https://fr.wikipedia.org/wiki/Pile_d%27ex%C3%A9cution). C'est de cette manière que la magie opère, pendant qu'un thread X réalisera un appel à une API par exemple et sera dans l'attente d'une réponse, le *main thread* pourra toujours exécuter d'autres méthodes comme rafraîchir la vue et écouter les évènements de l'utilisateur.

Il faut néanmoins garder en tête que les threads ne sont pas une solution miracle et ne possèdent pas de contrecoup. Certes, ils ne partagent pas la même pile d'exécution, mais tous les threads se partagent les mêmes ressources allouées au processus. Pour faire une analogie, plus vous invitez de gens à votre anniversaire, moins vous aurez de gâteau à manger. Les gens sont les threads, le gâteau, les ressources disponibles et vous, le *main thread*. Il ne faut pas en abuser sous peine de ralentir l'ensemble de votre application. 

Il est intéressant maintenant de nous pencher sur le fonctionnement des threads. Certaines règles régissent les threads et vont donc influencer notre manière de les utiliser dans notre application et notre développement.

En programmation orientée objet, un thread est représenté par l'instance d'une classe Thread, [comme celle-ci en JAVA](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.html). La classe expose des méthodes, comme `run()` qui va exécuter l'ensemble des instructions du thread ou bien la méthode `stop()` qui va permettre d'arrêter le thread peu importe son état (ceci est dangereux).

Du coup, pour réaliser un appel distant, on va pouvoir faire :

```kotlin
val newThread = object: Thread() { override fun run() = appelDistant() }.apply { start() }
```

`newThread` va donc s'exécuter et réaliser l'appel distant. Lorsqu'il aura terminé, il passera en état "terminé" et il pourra donc être détruit par le garbage collector dès qu'il n'y aura plus de références qui pointent sur son instance. À noter que dans cet exemple, le thread à l'origine de la création de `newThread` n'attend pas qu'il soit terminé pour continuer sa routine voire lui-même passé en état "terminé". Cependant, le thread ne pourra pas être détruit tant que `newThread` n'est pas en état "terminé".

### 1. Exemple d'un thread exécutant un autre et qui termine sans s'assurer que son enfant l'ait été

Penchons-nous sur un cas un plus complexe, où un thread `Parent thread` va créer un nouveau thread `Child thread` sans attendre qu'il termine pour terminer à son tour. Le `main thread` n'a qu'un rôle de superviseur ici, il initie le scénario et s'assure de ne pas terminer avant que tous les threads soient coupés grâce à la méthode `join()`.

```kotlin
// Implémentation de la classe RunnableThread
class RunnableThread(
    private val nameThread: String,
    private val block: () -> Unit
) : Thread() {
    override fun run() {
        println("$nameThread start")
        block.invoke()
        println("$nameThread stop")
    }
}

// Début du scope main()

private val threadList: MutableList<WeakReference<RunnableThread>> = ArrayList()

@JvmStatic
fun main(args: Array<String>) {
    println("Main thread start")

    val parentThread = RunnableThread("Parent thread") {
        // Running block of parent thread
        RunnableThread("Child thread") { /* Running block of child thread */ }
            .run {
                threadList.add(WeakReference(this))
                start()
            }
    }

    threadList.add(WeakReference(parentThread))
    parentThread.start()

    while (threadList.isNotEmpty()) {
        threadList.removeAt(0).get()?.join()
    }

    println("Main thread stop")
}
```

À vous de deviner l'ordre des print maintenant si vous avez compris !
<details>
  <summary><h5>Ordre des print [Spoiler]</h5></summary>
  
> 1. Main thread start
> 2. Parent thread start
> 3. Child thread start <-- interchangeable avec 4
> 4. Parent thread stop <-- interchangeable avec 3
> 5. Child thread stop
> 6. Main thread stop
>
> Cela vous étonne ? Primo, on ne sait pas exactement quand un thread sera exécuté même si on l'a instancié et que l'on a appelé la méthode `.start()`. Si le `main thread` ne faisait pas appel à la méthode `join()` de `Parent thread` via la liste `threadList`, il se serait terminé bien avant que `Parent thread` n'ait commencé. Du coup, il est possible que les positions 3. et 4. soient interchangées (et cela sera toujours correct). Secundo, `Parent thread` n'appelle lui jamais la méthode `join()` de son thread enfant ce qui fait qu'il n'attend pas qu'il ait terminé. Quant au `main thread`, il initie et boucle le scénario comme annoncé plus haut.
</details>

### 2. Exemple d'un thread exécutant un autre et qui s'assure que son enfant ait terminé avant de terminer à son tour

Voici un second scénario très similaire au premier :

```kotlin
@JvmStatic
fun main(args: Array<String>) {
    println("Main thread start")

    val parentThread = RunnableThread("Parent thread") {
        // Running block of parent thread
        RunnableThread("Child thread") { /* Running block of child thread */ }
            .apply {
                threadList.add(WeakReference(this))
                start()
            }
            // Parent is waiting child completion
            .join()
    }

    threadList.add(WeakReference(parentThread))
    parentThread.start()

    while (threadList.isNotEmpty()) {
        threadList.removeAt(0).get()?.join()
    }

    println("Main thread stop")
}
```

À vous de deviner l'ordre des print maintenant si vous avez compris !
<details>
  <summary><h5>Ordre des print [Spoiler]</h5></summary>
  
> 1. Main thread start
> 2. Parent thread start
> 3. Child thread start
> 4. Child thread stop
> 5. Parent thread stop
> 6. Main thread stop
>
> Cela vous étonne ? `Parent thread` appelle cette fois la méthode `join()` de son thread enfant ce qui fait qu'il attend qu'il ait terminé avant de terminer à son tour !
</details>

### 3. Exemple d'un thread executant un autre thread qui ne finit jamais

Un dernier pour la route, voici l'exemple le plus difficile :

```kotlin
@JvmStatic
fun main(args: Array<String>) {
    println("Main thread start")

    val parentThread = RunnableThread("Parent thread") {
        // Running block of parent thread
        RunnableThread("Child thread") { while (true) {} }
            .apply {
                threadList.add(WeakReference(this))
                start()
            }
    }

    threadList.add(WeakReference(parentThread))
    parentThread.start()

    while (threadList.isNotEmpty()) {
        threadList.removeAt(0).get()?.join()
    }

    println("Main thread stop")
}
```

À vous de deviner l'ordre des print maintenant si vous avez compris !
<details>
  <summary><h5>Ordre des print [Spoiler]</h5></summary>
  
> 1. Main thread start
> 2. Parent thread start
> 3. Child thread start <-- interchangeable avec 4
> 4. Parent thread stop <-- interchangeable avec 3
>
> Cela vous étonne ? `Child thread` étant dans une boucle `while (true)`, il ne peut jamais se terminer. La conséquence est que le `main thread` ne pourra jamais terminer à son tour. Cet exemple vise à sensibiliser sur les risques que peut générer un thread immortel, le plus courant étant une fuite mémoire car votre programme ne pourra jamais être terminé proprement sauf grâce à des mécanismes de nettoyage du système d'exploitation ou à un redémarrage physique de la machine.
</details>

L'ensemble de ces bouts de code est disponible sur ce répertoire dans les classes *ParentThreadWaitingThreadDeath*, *ParentThreadNotWaitingChildThreadDeath* et *ParentThreadNotWaitingChildThreadInfinityLiveDeath*.

### En synthèse

Ce qu'il faut retenir principalement des threads, c'est qu'ils ne sont pas magiques mais ils permettent de répondre au besoin de répartir les tâches entre la vue et les tâches plus longues (accès disque, accès réseau). Ils apportent une nouvelle complexité dans le code car il faut à la fois bien s'assurer qu'aucun thread ne soit immortel et il faut mettre en place des interfaces / protocoles pour réaliser des communications inter-threads (non vu dans les exemples). 

C'est dans ce contexte que les bibliothèques ReactiveX ont été créées. Bien entendu, il existe des solutions qui ont pu être proposées par les différents systèmes, Promise en JS ou AsyncTask puis Coroutine/Flow en Android à titre d'exemple.

## ReactiveX

Dans la programmation ReactiveX, on ne parle jamais directement de threads. On parle soit d'objets, **Observable**, soit de **Schedulers**. 
Nous allons commencer par aborder l'ensemble des objets Rx ! Comme leur nom l'indique, ce sont des objets qui vont émettre une valeur à des observers, voir [l'explication sur le patron observer](https://refactoring.guru/design-patterns/observer) pour comprendre rapidement ce que c'est. On distingue deux types de **Observable**, les **cold** et les **hot**. 

La différence entre les deux est très simple à retenir !
- un cold observable ne va pas émettre de valeur tant qu'il n'aura pas d'observers
- un hot observable va émettre des valeurs même s'il n'a pas d'observers
    
> Sur Android, un hot observable très connu est le LiveData<T>
    
Nous allons commencer par aborder les cold observables.
    
#### Cold observables
   
##### Observable
    
La classe de base de tous les objets Rx est **Observable\<T\>** (à distinguer de Observable de java.util). 
Tous les classes Rx héritent de celle-ci ! Comment il fonctionne ? Il va exécuter du code et mettre à disposition 3 méthodes à son observer.     
Le type \<T\> indique le type d'objet que l'instance de **Observable** va pouvoir émettre. 

| Type d'observable     | Observable  |
|-----------------------|-------------|
| Spécificité           | Peut émettre plusieurs fois la valeur <T> |
| Callback disponibles  | Il y a 3 callbacks disponibles avec un Observable<T>.<br> 1. onNext(\<T\> object). appelée dès lors qu'il y a une nouvelle valeur<br>2. onError(Throwable error), appelée dès lors que l'Observable rencontre une erreur<br>3. onComplete(), appelée dès lors que l'observable a terminé |

En RxJava, on peut le créer de plusieurs manières différentes un objet Rx. La manière la plus classique est d'utiliser la méthode statique `create<T>()`.
```kotlin
// De cette manière, c'est à nous de définir ce qu'il va se passer dans l'Observable.
// On peut imaginer par exemple l'appel à un web service paginé pour illustrer le propos
Observable.create<Response> { emitter ->  
    var currentPage = 0
  
    try { 
       while (currentPage < maxPage) {
          val response = appelDistant(currentPage)
          // Emission d'une nouvelle valeur                            
          emitter.onNext(response)                           
          currentPage++
       }                    
   } catch (e: HttpException) {
     // Emission d'une erreur                               
     emitter.onError(e)                              
   } finally {
     // Emission de la fin de l'Observable                               
     emitter.onComplete()                              
   }                             
}
```

## TO WRITE

NB : Sur Android, si le thread graphique met plus de deux secondes à réaliser sa routine, l'OS fait apparaître une pop-up ANR (Android Not Responding) à l'utilisateur qui peut donc fermer l'application en pensant qu'elle a crash
ReactiveX :
ReactiveX est un ensemble de bibliothèques qui permet de facilement exécuter du code de manière synchrone / asynchrone. La complexité d'utilisation est très forte, on peut presque parler d'un langage dans un langage au vu de ses spécificités propres. Vous trouverez les bibliothèques ReactiveX disponibles au lien suivant : 
Son intérêt est sa consistance sur la base théorique.En effet, malgré des différences de syntaxes propres aux différents langages, il est possible de comprendre du code Rx d'un autre langage grâce à ses connaissances et à sa propre expérience Rx. C'est également une communauté très active 


piège : interblocage, accès réseau / disque sur main thread, lecteur / exécuteur et bloc de code != paramètre

- les interblocages existent. Quand deux threads ont besoin d'une ressource et que A l'a verrouillé alors que B en a besoin et que A a besoin du traitement de B pour continuer, on appelle ça un interblocage (deadlock) car A ne relâche pas la ressource et B ne peut continuer sans. Conséquence : les threads ne mourront pas tant que le thread parent ne les coupe (sauf s'il est lui même concerné)
