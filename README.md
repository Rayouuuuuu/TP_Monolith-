# TP Spring Modulith & Kubernetes

Projet de monolithe modulaire avec **Spring Modulith 2.1**, **Spring Boot 4.1**, et son déploiement dans un cluster **Kubernetes (kind)** avec persistance **PostgreSQL**.

---

##  Structure du projet

```
demo-modulith-sb4/
├── src/
│   ├── main/
│   │   ├── java/org/formation/
│   │   │   ├── DemoModulithSb4Application.java
│   │   │   ├── catalog/                      # Module Catalog
│   │   │   │   ├── Catalog.java              # Service public
│   │   │   │   ├── CoffeeCreated.java        # Événement d'API public (record)
│   │   │   │   ├── CatalogController.java    # Contrôleur REST (/coffees)
│   │   │   │   └── internal/                 # Types internes cachés
│   │   │   │       ├── CoffeeEntity.java
│   │   │   │       └── CoffeeRepository.java
│   │   │   └── inventory/                    # Module Inventory
│   │   │       ├── InventoryController.java  # Contrôleur REST (/stocks/{id})
│   │   │       └── internal/                 # Types internes cachés
│   │   │           ├── StockEntity.java
│   │   │           ├── StockRepository.java
│   │   │           └── InventoryListener.java # Écouteur d'événement @ApplicationModuleListener
│   │   └── resources/
│   │       ├── application.properties        # Configuration locale (H2, Actuator)
│   │       └── application-k8s.properties    # Configuration cluster (PostgreSQL, ddl-auto=update)
│   └── test/
│       └── java/org/formation/
│           ├── ModularityTests.java          # Vérification des frontières modulaires
│           └── inventory/
│               └── InventoryModuleTests.java # Test isolé du module avec l'API Scenario
├── k8s/                                      # Manifests Kubernetes
│   ├── namespace.yaml                        # Namespace 'shop'
│   ├── deployment.yaml                       # Déploiement Modulith avec sondes et env
│   ├── service.yaml                          # Service Modulith (port 80 -> 8080)
│   ├── db-secret.yaml                        # Identifiants Postgres
│   ├── db-config.yaml                        # Configuration Postgres (URL JDBC)
│   ├── postgres-pvc.yaml                     # PersistentVolumeClaim (1Go)
│   └── postgres.yaml                         # Déploiement et Service PostgreSQL 16
├── kind-cluster.yaml                         # Configuration cluster Kind (1 control-plane, 2 workers)
├── Dockerfile                                # Build conteneur multi-stage en couches
└── pom.xml
```

---

##  Compilation et tests

### Lancer tous les tests
```bash
./mvnw clean test
```

### Vérifier la structure modulaire
```bash
./mvnw test -Dtest=ModularityTests
```

### Tester le module Inventory de manière isolée
```bash
./mvnw test -Dtest=InventoryModuleTests
```

---

## ☸️ Déploiement dans Kubernetes (kind)

### 1. Création du cluster
```bash
kind create cluster --name formation --config kind-cluster.yaml
```

### 2. Construction et chargement de l'image
```bash
./mvnw spring-boot:build-image -DskipTests
kind load docker-image demo-modulith-sb4:0.0.1 --name formation
```

### 3. Déploiement
```bash
# Namespace
kubectl apply -f k8s/namespace.yaml

# Base de données et stockage
kubectl apply -f k8s/db-secret.yaml
kubectl apply -f k8s/db-config.yaml
kubectl apply -f k8s/postgres-pvc.yaml
kubectl apply -f k8s/postgres.yaml

# Application Modulith
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

### 4. Accès local
```bash
kubectl port-forward -n shop svc/modulith 8080:80
```

---

##  Réponses aux questions des TPs

### Jour 1 — Le Monolithe Modulaire

1. **Pourquoi `catalog` ne dépend-il pas de `inventory` ?**  
   Pour découpler l'application et inverser la dépendance. `catalog` publie un événement public `CoffeeCreated` sans connaître ses consommateurs. `inventory` s'abonne à cet événement, ce qui permet d'ajouter d'autres abonnés sans toucher au code de `catalog`.

2. **Que change `@EnableAsync` sur le comportement du listener ?**  
   `@ApplicationModuleListener` intègre `@Async`. Sans `@EnableAsync`, le listener s'exécute de façon synchrone après le commit dans la même transaction HTTP. Avec `@EnableAsync`, il s'exécute dans un thread d'arrière-plan dédié.

3. **À quoi sert l'Event Publication Registry si le listener échoue ?**  
   Il implémente le pattern *Transactional Outbox*. L'événement est persisté dans la table `event_publication`. S'il échoue, il n'est pas marqué comme traité et peut être rejoué automatiquement (`spring.modulith.events.republish-outstanding-events-on-restart=true`) au redémarrage sans perte de données.

4. **En quoi cette organisation prépare-t-elle l'extraction en microservice ?**  
   Les frontières sont étanches (`*.internal.*`), les modules communiquent déjà par des contrats de données sérialisables (`CoffeeCreated`) et les entités en base sont découplées. L'extraction ne demande qu'à déporter les événements vers un broker externe (Kafka, RabbitMQ).

---

### Jour 2 — Le Modulith dans Kubernetes

1. **Pourquoi un Service ne connaît-il pas ses pods à l'avance ? Que se passe-t-il quand un pod est recréé ailleurs ?**  
   Les pods sont éphémères et changent d'adresse IP à chaque recréation. Le Service utilise un sélecteur de labels (`app: modulith`). Le contrôleur `EndpointSlice` surveille les labels et met à jour dynamiquement la liste des adresses IP saines dès qu'un pod redémarre.

2. **Pourquoi le tunnel port-forward ne permet-il pas de voir la répartition entre les pods ?**  
   `kubectl port-forward` est un tunnel direct vers un pod spécifique sélectionné à l'ouverture de la connexion ; il contourne le mécanisme d'équilibrage de charge de `kube-proxy`.

3. **Que se passerait-il si la sonde de vie (liveness) vérifiait l'accès à Postgres ?**  
   En cas de panne temporaire de la base, la sonde de vie échouerait sur tous les pods à la fois. Kubernetes redémarrerait tous les conteneurs en boucle (*cascading failure*), aggravant la panne. La sonde de vie ne doit surveiller que la santé interne du processus.

4. **Les 3 pods partagent la même base. Est-ce suffisant pour découper en plusieurs services ?**  
   Non. Pour découper proprement en microservices, il faut respecter le principe *Database-per-service* (une base par service), mettre en place un courtier de messages externe (Kafka, RabbitMQ) et séparer les manifests de déploiement.
