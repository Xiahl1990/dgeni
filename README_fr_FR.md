# Dgeni - Générateur de documentation [![Build Status](https://travis-ci.org/angular/dgeni.svg?branch=master)](https://travis-ci.org/angular/dgeni)

![](assets/dgeni-logo-600x400.png)

L'utilitaire de génération de documentation de node.js par angular.js et d'autres projets.

Dgeni se prononce comme le prénom féminin Jenny ([/ˈdʒɛni/](https://en.wikipedia.org/wiki/Help:IPA_for_English)),
à savoir le «d» est silencieux et le «g» est doux.

## Mise en route

Essayez [l'exemple de projet](https://github.com/petebacondarwin/dgeni-example) avec Dgeni. Ou vous cherchez peut-être un exemple qui [utilise AngularJS](https://github.com/petebacondarwin/dgeni-angular).

Regardez la présentation de Pete sur Dgeni (En anglais) :

[![ScreenShot](http://img.youtube.com/vi/PQNROxXajyQ/0.jpg)](http://youtu.be/PQNROxXajyQ)


## Documenter les applications Angular 1

Il existe deux projets qui s'appuient sur dgeni pour vous aider à créer la documentation pour les applications Angular 1 :

* dgeni-alive: https://github.com/wingedfox/dgeni-alive
* sia: https://github.com/boundstate/sia

Consultez-les et merci à [Ilya](https://github.com/wingedfox) et [Bound State Software](https://github.com/boundstate)
de nous avoir concocter ces projects.

## Installation

Vous aurez besoin de node.js et de plusieurs modules npm pour utiliser Dgeni. Récupérez node.js à partir d'ici :
http://nodejs.org/. Ensuite, dans le dossier de votre projet exécutez :

Dans le projet que vous souhaitez documenter, veuillez installez Dgeni en exécutant :

```
npm install dgeni --save-dev
```

Cela permet d'installer Dgeni et tous ses modules dépendants.


## Exécution de Dgeni

En fait Dgeni ne fait pas grand chose. Vous devez le configurez avec des **Packages** qui contiennent des **Services**
et des **Processeurs**. Ce sont les **Processeurs** qui convertissent réellement vos fichiers source
en fichiers de documentation.

Pour exécuter les processeurs, nous créons une nouvelle instance de `Dgeni`, en lui fournissant des **Packages**
à charger. Ensuite, il suffit d'appeler la méthode `generate()` sur cette instance. La méthode `generate()` exécute les
processeurs de manière asynchrone et renvoie une **Promise** qui renvoie le contenu des documents générés.

```js
var Dgeni = require('dgeni');

var packages = [require('./myPackage')];

var dgeni = new Dgeni(packages);

dgeni.generate().then(function(docs) {
  console.log(docs.length, 'docs générés');
});
```

### Exécution depuis la ligne de commande

Dgeni est normalement utilisé avec un outil de construction tels que Gulp ou Grunt, mais il peut être utilisé également
avec un outil de ligne de commande.

Si vous installez Dgeni globalement, alors vous pouvez l'exécuter de n'importe où :

```bash
npm install -g dgeni
dgeni some/package.js
```

Si Dgeni n'est installé que localement, alors vous devez spécifier explicitement le chemin :

```bash
npm install dgeni
node_modules/.bin/dgeni some/package.js
```

ou vous pouvez exécuter l'outil dans un script npm :

```js
{
  ...
  scripts: {
    docs: 'dgeni some/package.js'
  }
  ...
}
```


L'utilisation est le suivant :


```bash
dgeni chemin/du/packagePrincipal [chemin/pour/unAutre/package ...] [--log level]
```

Vous devez fournir le chemin pour charger un ou plusieurs Packages Dgeni. Vous pouvez
définir le niveau de journalisation (facultatif).


## Packages

Les **Services**, les **Processeurs**, les valeurs de configuration et les templates sont regroupés dans un `Package`. Les Packages
peuvent dépendre de d'autres Packages. De cette façon, vous pouvez construire votre configuration personnalisée
par dessus une configuration existante.

### Définition d'un Package

Dgeni fournit un constructeur de `Package` pour créer de nouveaux Packages. Une instance de Package a des méthodes pour enregistrer des **Services** et
des **Processeurs** et permet de configurer les propriétés des **Processeurs** :

```js
var Package = require('dgeni').Package;
var myPackage = new Package('myPackage', ['packageDepencency1', 'packageDependency2']);

myPackage.processor(require('./processors/processor1'));
myPackage.processor(require('./processors/processor2'));

myPackage.factory(require('./services/service1'));
myPackage.factory(require('./services/service2'));

myPackage.config(function(processor1, service2) {
  service2.someProperty = 'some value';
  processor1.specialService = service2;
});
```


## Services

Dgeni utilise énormément **l'injection de dépendance (DI)** pour  instancier les objets. Les objets qui
seront instanciés par le système de DI, doivent être fournis par une **fonction factory**, qui est enregistrées
dans un **Package**, soit comme un **Processeur** par `myPackage.processor(factoryFn)`, ou soit comme un **Service**
par `myPackage.factory(factoryFn)`.

### Définition d'un Service

Les paramètres d'une fonction factory sont des dépendances sur d'autres services que le système de DI doit trouver
ou instancier et fournir à la fonction factory.

**car.js** :
```js
module.exports = function car(engine, wheels) {
  return {
    drive: function() {
      engine.start();
      wheels.turn();
    }
  };
});
```

Ici, nous avons défini un service `car`, qui dépend de deux autres services, `engine` et `wheels`
définis ailleurs. Notez que ce service `car` ne se soucie pas comment et où ces dépendances sont
définies. Il s'appuie sur le système de DI pour leur fournir en cas de besoin.

Le service `car` renvoyé par la fonction factory est un objet contenant une méthode, `drive()`,
qui à son tour appelle des méthodes sur `engine` et `wheels`.


### Enregistrement d'un Service

Vous pouvez ensuite enregistrer le service avec un Package :

**myPackage.js**:
```jsv
var Package = require('dgeni').Package;

module.exports = new Package('myPackage')
  .factory(require('./car'));
```

Ce Service de car est maintenant disponible pour n'importe quel autre bloc de service, de processeur ou de configuration :

```js
var Package = require('dgeni').Package;

module.exports = new Package('greenTaxiPackage', ['myPackage'])

  .factory(function taxi(car) {
    return {
      orderTaxi: function(place) { car.driveTo(place); }
    };
  })

  .config(function(car) {
    car.fuel = 'hybrid';
  });
```


## Processeurs

Les **Processeurs** sont des **Services** qui contiennent une méthode `$process(docs)`. Les processeurs sont exécutés
les uns après les autres dans un pipeline. Chaque processeur prend la collection de documents depuis le processeur précédent
et la manipule, peut-être pour insérer de nouveaux documents ou ajouter des méta-données aux documents qui sont
déjà présents.

Les processeurs peuvent avoir des propriétés qui expliquent à Dgeni quand ils doivent être exécutés dans le pipeline et
la façon de valider la configuration du processeur.

* `$enabled` - si défini à `false` alors ce Processeur ne sera pas inclus dans le pipeline
* `$runAfter` - un tableau de strings, où chaque string est le nom d'un processeur qui doit figurer
dans le pipeline **avant** ce Processeur
* `$runBefore` - un tableau de strings, où chaque string est le nom d'un processeur qui doit figurer
dans le pipeline **après** ce Processeur
* `$validate` - un objet de [http://validatejs.org/](http://validatejs.org/) qu'utilise Dgeni
pour valider les propriétés de ce processeur.


**Notez que la fonctionnalité de validation a été déplacé dans son propre Package Dgeni : `processorValidation`.
Actuellement dgeni ajoute automatiquement ce nouveau package dans une nouvelle instance de dgeni afin qu'il soit toujours disponible
pour être compatible avec les versions antérieures. Dans une prochaine version, ce packge sera déplacé dans `dgeni-packages`.**

### Définition d'un Processeur

Vous définissez les Processeurs comme vous le feriez pour un Service :

**myDocProcessor.js**:
```js
module.exports = function myDocProcessor(dependency1, dependency2) {
  return {
    $process: function (docs) {
        //... faire des choses avec les docs ...
    },
    $runAfter: ['otherProcessor1'],
    $runBefore: ['otherProcessor2', 'otherProcessor3'],
    $validate: {
      myProperty: { presence: true }
    },
    myProperty: 'une valeur de configuration'
  };
};
```


### Enregistrement d'un Processeur

Vous pouvez ensuite enregistrer le processeur avec un Package :
**myPackage.js**:
```jsv
var Package = require('dgeni').Package;

module.exports = new Package('myPackage')
  .processor(require('./myDocProcessor'));
```

### Traitement asynchrone

La méthode `$process(docs)` peut être synchrone ou asynchrone :

* Si elle est synchrone, elle devra retourner `undefined` ou un nouveau tableau de documents.
Si elle retourne un nouveau tableau de documents alors ce tableau remplacera le précédent tableau `docs`.
* Si elles est asynchrone, alors elle doit retourner une **Promise**, qui permettra de résoudre (resolve) `undefined`
ou une nouvelle collection de documents. En retournant une **Promise**, le processeur dit à Dgeni
qu'il est asynchrone et Dgeni attendra la promise pour résoudre avant d'appeler le
prochain processeur.


Voici un exemple d'un **Processeur** asynchrone
```js
var qfs = require('q-io/fs');
module.exports = function readFileProcessor() {
  return {
    filePath: 'some/file.js',
    $process(docs) {
      return qfs.readFile(this.filePath).then(function(response) {
        docs.push(response.data);
      });
    }
  };
```

### Les Packages Dgeni Standard

Le [dépôt dgeni-packages](https://github.com/angular/dgeni-packages) contient plusieurs Processeurs -
de première nécessité au plus complexe spécifique à angular.js. Ces processeurs sont regroupés dans des Packages :

* `base` -  contient des processeurs basiques de lecture et d'écriture de fichier, ainsi qu'un processeur
de rendu abstrait.

* `jsdoc` - dépend de `base` et ajoute des processeurs et des services pour supporter l'analyse et
l'extraction de balise de commentaires jsdoc dans le code.

* `typescript` - dépend de `base` et ajoute des processeurs et des services pour supporter l'analyse et
l'extraction des balises de style jsdoc depuis les commentaires du code en TypeScript (*.ts).

* `nunjucks` - fournit un moteur de rendu bsé sur
[nunjucks](http://mozilla.github.io/nunjucks/).

* `ngdoc` - dépend de `jsdoc` et `nunjucks` et ajoute un traitement supplémentaire pour les
extensions d'AngularJS à jsdoc.

* `examples` - dépend de `jsdoc` et fournit les processeurs pour extraire des exemples des commentaires
de jsdoc et les convertir en fichiers qui peuvent être exécutés.

* `dgeni` - support pour la documentation des Packages de dgeni.


### Processeurs pseudo marqueurs

Vous pouvez définir des processeurs qui ne font rien mais qui agissent comme des marqueurs pour les différentes étapes
du traitement. Vous pouvez utiliser ces marqueurs dans les propriétés `$runBefore` et `$runAfter` pour s'assurer que votre
processeur soit lancé au bon moment.

Les ****Packages** dans dgeni-packages définissent certains processeurs marqueurs. Voici la liste
dans l'ordre que Dgeni les ajoutera à la pipeline du traitement :


* reading-files (lecture des fichiers) *(défini dans base)*
* files-read (fichiers lus) *(défini dans base)*
* parsing-tags (analyse des balises) *(défini dans jsdoc)*
* tags-parsed (balises analysées) *(défini dans jsdoc)*
* extracting-tags (extraction des balises) *(défini dans jsdoc)*
* tags-extracted (balises extraites) *(défini dans jsdoc)*
* processing-docs (traitement des documents) *(défini dans base)*
* docs-processed (documents traités) *(défini dans base)*
* adding-extra-docs (ajout des documents supplémentaires) *(défini dans base)*
* extra-docs-added (documents supplémentaires ajoutés) *(défini dans base)*
* computing-ids (détermination d'un id) *(défini dans base)*
* ids-computed (Id déterminé) *(défini dans base)*
* computing-paths Détermination des chemins *(défini dans base)*
* paths-computed Chemins déterminés *(défini dans base)*
* rendering-docs (rendu des documents) *(défini dans base)*
* docs-rendered (documents rendus) *(défini dans base)*
* writing-files (écriture des fichiers) *(défini dans base)*
* files-written (fichiers écrits) *(défini dans base)*


## Configuration des Blocs

Vous pouvez configurer les **Services** et les **Processeurs** définis dans un **Package** ou ses dépendances
en enregistrant des **blocs de configuration** avec le **Package**. Ce sont des fonctions qui peuvent être
injectées avec des **Services** et des **Processors** par le système de DI. Ceci vous donne la possibilité de
de définir, sur eux, des propriétés.


### Enregistrement d'un bloc de configuration

Vous enregistrez un **bloc de configuration** en appelant `config(configFn)` sur un Package.

```js
myPackage.config(function(readFilesProcessor) {
  readFilesProcessor.sourceFiles = ['src/**/*.js'];
});
```

## Evénements de Dgeni

Dans Dgeni vous pouvez déclencher et gérer des **évènements** pour permettre aux packages de participer au cycle
du traitement de la génération de la documentation.


### Déclenchement des évènements

Vous déclenchez un événement tout simplement en appelant `triggerEvent(eventName, ...)` sur une instance `Dgeni`.

Le `eventName` est une chaîne qui identifie l'événement qui doit être déclenché, ce qui permet de connecter
des gestionnaires d'événements. Les arguments supplémentaires sont passés dans les gestionnaires.

Chaque gestionnaire qui est enregistré pour l'événement est appelé en série. La valeur de retour
de l'appel est une promise à l'événement géré. Cela permet à des gestionnaires d'événements d'être asynchrone.
Si un gestionnaire retourne une promise rejetée, l'événement déclencheur est annulé et la promise rejeté
est retourné.

Par exemple:

```js
var eventPromise = dgeni.triggerEvent('someEventName', someArg, otherArg);
```

### Gestion des évènements

Vous enregistrez un gestionnaire d'événement dans un `Package`, en appelant `handleEvent(eventName, handlerFactory)` sur
l'instance du package. Le handlerFactory sera utilisé par le système DI (injection de dépendance) pour récupérer le gestionnaire, ce qui vous
permet d'injecter des services qui seront disponibles pour le gestionnaire.

La factory de gestionnaire devrait retourner la fonction du gestionnaire. Cette fonction recevra tous les arguments passés
à la méthode `triggerHandler. Au minimum, elle aura `eventName`.

Par exemple:

```js
myPackage.eventHandler('generationStart', function validateProcessors(log, dgeni) {
  return function validateProcessorsImpl(eventName) {
    ...
  };
});

```

### Evénements intégrés

Dgeni déclenche lui-même les événements suivants au cours de la génération de documentation :

* `generationStart` : déclenché après que l'injecteur ait été configuré et avant que les processeurs
  commencent leur travail.
* `generationEnd`: déclenché après que les processeurs aient tous terminé leur travail avec succès.
* `processorStart`: déclenché juste avant l'appel de `$process` pour chaque processeur.
* `processorEnd`: déclenché juste après que `$process` soit terminé avec succès pour chaque processeur.
