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

En synthèse, le *main thread* joue un rôle majeur dans l'exécution de l'application car si jamais il doit réaliser une opération longue ou se retrouver bloqué avec un `while(true)` par exemple, l'application ne répondrait plus aux évènements utilisateurs.

> Sur Android, si le thread graphique met plus de deux secondes à réaliser sa routine de rafraîchissement de la vue, l'OS fait apparaître une pop-up ANR (Android Not Responding) à l'utilisateur qui peut donc fermer l'application en pensant qu'elle a crash alors que cette dernière réalise simplement un appel distant ou un algorithme complèxe.

En lisant ces deux dernières lignes, on se rend donc compte qu'il est primordial d'offrir un maximum de temps et de ressources au rafraîchissement de la vue afin d'assurer la meilleure expérience utilisateur possible. 

> C'est très facile de dire ça, mais comment fait-on pour réaliser des appels distants, des requêtes en base de données, des algorithmes longs et complexes si je ne peux pas les exécuter sur le thread principal ?

Pour pouvoir réaliser des opérations longues ou coûteuses en ressource, nous pouvons nous appuyer sur les threads. Pour rappel, un processus peut contenir un à N threads, et chacun des threads possède sa propre [pile d'exécution](https://fr.wikipedia.org/wiki/Pile_d%27ex%C3%A9cution). C'est de cette manière que la magie opère, pendant qu'un thread X réalisera un appel à une API par exemple et sera dans l'attente d'une réponse, le *main thread* pourra toujours exécuter d'autres méthodes et écouter les évènements.

Il faut néanmoins garder en tête que les threads ne sont pas une solution miracle et ne possèdent pas de contrecoup. Certes, ils ne partagent pas la même pile d'exécution, mais tous les threads se partagent les mêmes ressources allouées au processus. Pour faire une analogie, plus vous invitez de gens à votre anniversaire, moins vous aurez de gâteau à manger. Les gens sont les threads, le gâteau, les ressources disponibles. Il ne faut pas en abuser sous peine de ralentir l'ensemble de votre application. 

Il est intéressant maintenant de nous pencher sur le fonctionnement des threads. Certaines règles régissent les threads et vont donc influencer notre manière de les utiliser dans notre application et notre développement.

En programmation orientée objet, un thread est représenté par l'instance d'une classe Thread, [comme celle-ci en JAVA](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.html). La classe expose des méthodes, comme `run()` qui va exécuter l'ensemble des instructions du thread ou bien la méthode `stop()` qui va permettre d'arrêter le thread peu importe son état (ceci est dangereux).

Du coup, pour réaliser un appel distant, on va pouvoir faire :

```kotlin
val thread = object: Thread() { override run() = appelDistant() }

thread.start()
```

Le thread va donc s'exécuter et réaliser l'appel distant. Lorsqu'il aura terminé, il sera détruit comme tout objet dès lors qu'il n'y a plus de référence forte vers lui et qui ne possède plus aucune référence 

Théorie développement :- Développement front dans un système => un processus- Un processus est composé d'un thread principal que l'on appelle le main thread ou UI thread en anglais- Ce thread est en charge du rafraîchichissement de la vue, des évènements, des animations, bref à beaucoup de choses où il faut être rapide
Théorie système :- Tous les processus se partagent les ressources du système- Un processus partage les ressources allouées par le système entre ses différents threads- Plus un processus a de threads, moins ses threads iront vite (analogie, plus vous avez d'invités à votre anniversaire, moins vous aurez de gâteau à manger.)
- Le principe de parent existe chez les threads- Un thread principal peut se terminer sans que ses threads enfants le soient, on les appelle donc des threads orphelins car plus personne ne va les couper / attendre qu'ils se terminent.
- les interblocages existent. Quand deux threads ont besoin d'une ressource et que A l'a verrouillé alors que B en a besoin et que A a besoin du traitement de B pour continuer, on appelle ça un interblocage (deadlock) car A ne relâche pas la ressource et B ne peut continuer sans. Conséquence : les threads ne mourront pas tant que le thread parent ne les coupe (sauf s'il est lui même concerné)
Conclusion :
Pour réaliser des opérations coûteuses en temps ou en ressources, il vaut mieux s'appuyer sur des threads autre que le thread principal. Ainsi, un maximum de temps est dédié au rafraîchissement de la vue et aux animations, l'application reste fluide et Interactive pour l'utilisateur.
NB : Sur Android, si le thread graphique met plus de deux secondes à réaliser sa routine, l'OS fait apparaître une pop-up ANR (Android Not Responding) à l'utilisateur qui peut donc fermer l'application en pensant qu'elle a crash
ReactiveX :
ReactiveX est un ensemble de bibliothèques qui permet de facilement exécuter du code de manière synchrone / asynchrone. La complexité d'utilisation est très forte, on peut presque parler d'un langage dans un langage au vu de ses spécificités propres. Vous trouverez les bibliothèques ReactiveX disponibles au lien suivant : 
Son intérêt est sa consistance sur la base théorique.En effet, malgré des différences de syntaxes propres aux différents langages, il est possible de comprendre du code Rx d'un autre langage grâce à ses connaissances et à sa propre expérience Rx. C'est également une communauté très active 
