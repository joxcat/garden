---
{"dg-publish":true,"permalink":"/cpe/projet-scientifique/"}
---


#CPE 

## Objectifs
Réaliser d'une part un simulateur d'incendie permettant la création, le suivi et la propagation de feux de différents types (localisés sur une carte), et d'autre part de créer un dispositif de gestion de services d'urgences permettant, à partir d'informations collectées par des capteurs, de déployer et gérer les dispositifs adaptés pour éteindre les incendies. 

## TODO
- [ ] Liste tests (unitaires & E2E) IOT
- [ ] Diagramme classe et séquence
- [ ] Protocole réseau
- [ ] Structure de projet

## Jalons
- [ ] TD: Présentation architecture globale et gestion du projet 📅 2022-12-08
- [ ] TD: Présentation conception logicielle. **Diagramme de classe / séquence** et Schéma BD 📅 2022-12-13
- [ ] TD: Présentation / démo chaîne IOT 📅 2023-01-04
	- [ ] Démo collecte, envoi et réception des données des feux dans la ville ainsi que génération d'appel REST.
- [ ] TD: Soutenance + démos finales 📅 2023-01-13

## Pourquoi du Rust en IOT est pertinent ?
- Support aisé de nouvelles plateformes si la target est supportée (no_std)
- Ref. RustConf 2022 contrôle des trains https://youtu.be/qaj5q88eLjk
- Un compilateur qui aide à éviter toutes les erreurs d'allocations / d'accès indexés
- Une création et implémentation de protocole simplifié grâce à [serde](https://lib.rs/serde) et [nom](https://lib.rs/nom)
- Documentation spécialisée pour le MicroBit ([MicroRust](https://droogmic.github.io/microrust) [Librairie microbit V1](https://lib.rs/crates/microbit))
- Librairie basée sur une [couche d'abstraction](https://github.com/nrf-rs/nrf-hal) pour les appareils NRF
- Compatible avec [l'écosystème d'outils Knurling](https://knurling.ferrous-systems.com/tools/) 
- Radio examples : [nrf-rs/microbit/pull/90](https://github.com/nrf-rs/microbit/pull/90/files)

## Conception IOT
### Structures initiale des données simulées
- Matrice `6x10` capteurs
- Valeur intensité du feu `0-9` 
- `(col, line, intensity)`

### Données capteur
| Donnée      | Besoin                                      | Solution                        |
| ----------- | ------------------------------------------- | ------------------------------- |
| Identifiant | Identifier unicité                          | ID dépendant du concentrateur                  |
| Latitude    | Localisation                                | Ligne matrice                   |
| Longitude   | Localisation                                | Colonne matrice                 |
| Température | Identifier feu                              | stockage température du capteur |
| Type        | Différencier capteur & relai dans le réseau | Enum type de device             |

### Calcul de l'ID
- Envoie d'une requête de demande d'ID au concentrateur (ref DHCP) en utilisant l'id physique du capteur
- Les capteurs sur le chemin stockent l'id physique temporairement en attendant l'id interne (aucun broadcast n'est effectué), TTL de 15s
- Le concentrateur regarde sa table de capteurs et génère un ID pour le capteur puis le renvoie au capteur qui lui a relayé l'id
-  La chaine de capteur rajoute l'id logique dans la table des capteurs forward au capteur suivant puis supprime l'id physique de la table temporaire
- Le capteur initial se débloque et enregistre son id du réseau

#### Cas particulier pas de relai ?
- Manager = Plus petit ID de mesh
- Si pas d'ID de mesh, plus petite adresse physique

### Trust on first use & chiffrement
- Curve25519 ? => Clé 128bits

#### Données réseau capteur
- Etat du capteur
	- Connecté au relai directement
	- Connecté au relai indirectement
	- Connecté au réseau de capteurs
	- Seul
- Table des capteurs (ref ARP) - tri par distance, puis par charge (plus faible), puis par ID de mesh
	- Distance en hop avec le capteur
	- TTL du capteur (60s par défaut ?)
	- Distance en hop avec le relai
	- Charge du capteur / relai (nombre de capteurs connectés)
	- 

### Proto radio / UART
Objectifs : Mesh, résistance aux interférences, check is alive
Les broadcast physiques ne sont pas repartagés
Les broadcast logiques sont repartagés

### Découpage trame
> https://lancaster-university.github.io/microbit-docs/ubit/radio/#capabilities
> Typically 32 bytes, but reconfigurable in code up to 1024 bytes.


### Différentes requêtes

| Request ID | Request Abbrv | Connection           | Ident               | Description                                            |
| ---------- | ------------- | -------------------- | ------------------- | ------------------------------------------------------ |
| 0          | `HELOP`       | Broadcast            | Physical            | Utilisé pour découvrir l'état initial du réseau local  |
| 1          | `OLEHP`       | Targeted             | Logical => Physical | Partage initial de la table de machines                |
| 2          | `IDENT`       | Targeted             | Physical => Logical | Demande au manager de mesh un identifiant de mesh      |
| 3          | `TNEDI`       | Targeted             | Logical => Physical | Assigne un identifiant de mesh                         |
| 4          | `TRUSTB`      | Broadcast (depth -1) | Logical             | Partage la clef publique du manager                    |
| 5          | `TRUSTT`      | Targeted             | Logical => Logical  | Partage la clef publique de la machine                 |
| 6          | `HELOL`       | Broadcast (depth 1)  | Logical             | Met a jour de manière forcée la table des machines     |
| 7          | `OLEHL`       | Targeted             | Logical => Logical  | Partage de la table de machines si adresse est connue  |
| 8          | `PUSH`        | Targeted             | Logical => Logical  | Envoie une mise à jour d'un capteur                    |
| 9          | `PUSH_ACK`    | Targeted             | Logical => Logical  | Confirme que les données ont correctement été envoyées |
| 10         | `ADENY`       | Targeted             | Logical => Physical | Accès non autorisé                                    |
| 11         | `UDENY`       | Targeted             | Logical => Logical  | Inconnu dans la table d'identifiants                   |

#### Chiffrement des requêtes
| Request Abbrv | Partie chiffrée   |
| ------------- | ----------------- |
| `HELOP`       | Clair             |
| `OLEHP`       | Clair             |
| `IDENT`       | Clair             |
| `TNEDI`       | Clair             |
| `TRUST`       | Clair             |
| `HELOL`       | Clair             |
| `OLEHL`       | Clair             |
| `PUSH`        | Données chiffrées |
| `PUSH_ACK`    | Données chiffrées |
| `ADENY`       | Clair             |
| `UDENY`       | Clair             |

### Proxy UART / API