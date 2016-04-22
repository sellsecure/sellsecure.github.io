---
title: 'Asynchroniser ses traitements PHP avec Beanstalkd'
---
Le cœur de SellSecure est une "machine" à scorer des transactions. Pour ce faire nous récupérons des dizaines de datas de nos commerçants et nous les passons dans un moteur de règles.
Le moteur va chercher dans nos bases de données les récurrences des datas et pour chaque type de donnéees il va lui appliquer une pondération. Pour illustrer, on va aller chercher combien de fois est apparu le mail du client sur les n derniers mois. Suivant le nombre de fois où cela apparaît nous modifions la pondération, ce qui influe sur le score final de la transactions.

## La problématique

Ces traitements qui récupèrent de l'historique sont longs, trop longs pour un traitement web. On est pas loin de la dizaine de secondes avant de pouvoir rendre un résultat.

Le traitement se faisant après le paiement des clients sur le site de nos commerçants ils seraient idiots de les faire patienter aussi longtemps. Nous avons donc du mettre en place une asynchronisation de ces traitements. 
De cette façon on redonne  la main au commerçant rapidement et il récupère l'information quand elle est disponible.

## La solution technique

C'est une problématique relativement classique en informatique, du coup on a de la chance les solutions sont nombreuses:
 - RabbitMQ
 - ZeroMQ
 - Beanstalkd
 - Reddis 
 - …
 - 
Nous avons fait notre choix en fonction de notre besoin, nous avions besoin de gestionnaire de files d'attentes rien de plus. 
Sur le modèle Unix nous avons donc choisi Beanstalkd qui ne fait que ça et qui le fait bien!

## Beanstalkd
https://kr.github.io/beanstalkd/

> Beanstalk is a simple, fast work queue.

Beanstalkd est fait en C, donc pas de dépendance. Il ne fait que de la file d’attente il est super rapide, stable et la persistance est possible, que demander de plus !

Pour le câbler sur un projet PHP on l’utilise avec pheanstlakd → https://github.com/pda/pheanstalk
Idem, c’est simple du coup pas besoin de mettre en place un bundle juste une ligne dans le composer.json

    composer require pda/pheanstalk

## Comment on met ça dans du code PHP ?

Pour créer et utiliser une file d’attente Beanstalkd il faut qu’il soit installé sur le serveur:

    yum install beanstalkd

Ensuite dans notre code on lui dit où écouter:

    use Pheanstalk\Pheanstalk;

    $pheanstalk = new Pheanstalk('127.0.0.1');

On déclare une file et on la remplit avec nos messages:

    $pheanstalk
     ->useTube('testtube')
     ->put("job payload goes here\n");

 
 Et c'est tout !  Le `useTube` sert à la fois à créer la file et à lui dire que l'on va mettre les messages dedans. 
Traiter les files

Pour traiter les files c’est aussi simple:

    $job = $pheanstalk
     ->watch('testtube')
     ->reserve();

Je regarde dans testTube et dès qu’un truc passe, je le reserve. Puis je le traite

    echo $job->getData();

Enfin je le vire de la file:

    $pheanstalk->delete($job);

Et ainsi de suite, dans une boucle while(true) comme suit

    $pheanstalk = new Pheanstalk('127.0.0.1');
        while (true) {
            $pheanstalk->watch("testtube");  
            …
        }

Vous remarquerez que j'ai mis un joli `if statstube` cela permet de dédié des workers à certains tubes

## Isoler les tubes et les traitements

Jusqu'ici nous avons un seul tube, hors nous avons besoin d'isoler les traitements entre eux. Rien de plus simple on va créer un `worker` par type de job. Et on va écouter le tube qui correspond en priorité.

     $pheanstalk = new Pheanstalk('127.0.0.1');
        while (true) {
            $pheanstalk->watch("prioTube");
    
            if ($pheanstalk->statsTube('prioTube')["current-jobs-ready"] > 0) {
                $job = $pheanstalk->reserve();
                // Je fais mon traitement
                $pheanstalk->delete($job);
            } else {
                $pheanstalk->watch("testTube");
                $job = $pheanstalk->reserve();
                // Je fais mon traitement
                $pheanstalk->delete($job);
            }
        }

De cette manière nos workers traitent les jobs prioritaires en priorité et si le tube est vide alors ils vont chercher les jobs du testtube.

Voilà vous savez tout pour construire vos premières files d'attente avec Beanstalkd !


