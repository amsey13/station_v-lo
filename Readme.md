# Simulation de Location de Vélo en Libre-Service

## 1. Binôme et introduction
**Binôme :**  
- Kadiatou Barry  
- Mamady Mansare

**Introduction :**  
Ce projet simule un système de location de vélos en libre-service avec un centre de contrôle supervisant les stations et la flotte de vélos.  
La simulation inclut la gestion des utilisateurs, des états de vélos, des accessoires, des maintenances et des vols.



## 2. HowTo

### 2.1 Récupérer les sources depuis le depôt git
```bash
git clone <https://github.com/amsey13/coo.git>
cd projet_coo
```
### 2.2 Générer la documentation
```bash
make doc
```
Toute la documentation sera dans le dossier doc à la racine du projet
### Compiler et exécuter les sources
```bash
make compile
make run
```

### 2.4 Compiler et exécuter les tests
```bash
make test
```
### 2.5 Générer et exécuter l’archive (.jar)
```bash
make jar
```
## 3. Conception & Architecture

### 3.1 Architecture principale

Le système est organisé autour de plusieurs blocs principaux, avec une séparation claire des responsabilités :

- **Véhicules**
  - Hiérarchie abstraite avec des types concrets comme `Bike` et `Scooter`.
  - Chaque véhicule possède un coût de location, un compteur de locations, et un état (disponible, loué, hors service, volé).
  - Les accessoires sont ajoutés dynamiquement grâce au pattern Decorator (ex. panier, électrification).

- **Stations et emplacements (slots)**
  - Une `Station` gère un ensemble d’`Slot` (composition forte : une station “possède” ses emplacements).
  - Chaque `Slot` peut contenir au plus un véhicule et compte le nombre de “tours seul” pour la logique de vol.
  - La station sait si elle est vide, pleine, ou en situation déséquilibrée (trop de vélos / pas assez) et notifie le centre de contrôle via le pattern Observer.

- **Utilisateurs**
  - Un `User` possède un identifiant, un solde et au plus un véhicule loué.
  - Il ne peut louer un véhicule que si son solde permet de payer le coût complet (véhicule + accessoires).
  - Le même objet `Vehicle` est conservé côté utilisateur puis restitué en station, ce qui permet de suivre précisément son historique d’utilisation.

- **Centre de contrôle**
  - Le `ControlCenter` observe les stations, maintient un historique des événements et déclenche la redistribution lorsque certaines stations restent vides ou pleines trop longtemps.
  - Il délègue l’algorithme de redistribution à une stratégie (Strategy pattern), ce qui permet de changer le comportement global sans modifier les stations.

- **Moteur de simulation**
  - La classe `Simulation` fait évoluer le système par “tours” discrets.
  - À chaque tour :
    1. Des actions aléatoires sont effectuées sur les stations (dépôts et locations automatiques),
    2. Les utilisateurs louent ou rendent des véhicules,
    3. Les vols potentiels sont détectés,
    4. La maintenance progresse,
    5. Le centre de contrôle peut déclencher une redistribution suivant la stratégie configurée.

Cette architecture respecte le principe de responsabilité unique (SRP) et facilite l’extension du système (ajout de nouveaux véhicules, d’accessoires, de stratégies, etc.).

---

### 3.2 Design Patterns utilisés

- **State**
  - Les véhicules ont un état (`Available`, `Rented`, `OutOfService`, `Stolen`) qui encapsule les comportements possibles.
  - Chaque état gère ses propres transitions (par exemple, un véhicule volé ne peut plus redevenir disponible).
  - Cela évite les grosses conditions `if/else` et rend le cycle de vie des véhicules plus clair et extensible.

- **Strategy**
  - L’algorithme de redistribution est représenté par l’interface `RedistributionStrategy`.
  - Des implémentations comme `RoundRobinStrategy` ou `RandomRobinStrategy` proposent différents comportements.
  - Le `ControlCenter` référence une stratégie et lui délègue la décision de redistribution, ce qui permet de changer l’algorithme sans modifier le reste du code.

- **Observer**
  - Les `Station` jouent le rôle de sujets observables, le `ControlCenter` celui d’observateur.
  - À chaque dépôt ou location, la station notifie le centre de contrôle.
  - Cela découple la logique locale de la station de la supervision globale et respecte le principe d’inversion des dépendances.

- **Decorator**
  - Les accessoires (`Basket`, `Electrification`, etc.) sont implémentés comme des décorateurs autour d’un `Vehicle`.
  - Chaque décorateur délègue au véhicule sous-jacent mais ajoute son propre comportement (surcoût, enrichissement de la description).
  - Plusieurs décorateurs peuvent être empilés sur un même véhicule sans multiplier les sous-classes (ex. vélo électrique avec panier).

- **Factory Method**
  - Les factories (par exemple `BikeFactory`) encapsulent la création de véhicules.
  - Toute la logique d’initialisation est centralisée, ce qui simplifie le code client (simulation, tests) et facilite l’ajout de nouveaux types de véhicules.

- **Visitor**
  - Les opérations comme la peinture ou la réparation sont modélisées avec des visiteurs (`Painter`, `Repairer`).
  - Les véhicules acceptent un visiteur via `accept(Visitor)`, ce qui permet d’ajouter de nouvelles opérations sans modifier la hiérarchie des véhicules.
  - Ce pattern illustre le double dispatch (le comportement dépend à la fois du type du visiteur et du type du véhicule).

---

### 3.3 Choix d’implémentation

- **Persistance des objets loués**
  - Un utilisateur stocke une référence directe vers l’objet `Vehicle` loué (et non uniquement un identifiant).
  - Lors de la restitution, c’est exactement le même objet qui est remis en station, avec son compteur de locations mis à jour.
  - Cela permet d’atteindre correctement la limite de locations et de déclencher l’état `OutOfService` puis la maintenance.

- **Gestion du temps et des vols**
  - La simulation avance via un entier `currentTime` qui représente le nombre de tours.
  - Chaque `Slot` garde un compteur de tours où le véhicule est seul (`aloneTurns`).
  - Si un véhicule reste seul pendant un nombre donné de tours (par exemple 2), il est considéré comme volé :
    - son état passe à `Stolen`,
    - il est retiré de la station.

- **Maintenance**
  - Lorsque le nombre de locations dépasse un seuil (par exemple 20), le véhicule passe en état `OutOfService`.
  - Un visiteur `Repairer` peut alors démarrer une maintenance de durée fixe (nombre de tours).
  - Pendant cette période, le véhicule ne peut pas être loué. Une fois la maintenance terminée, il redevient disponible.
