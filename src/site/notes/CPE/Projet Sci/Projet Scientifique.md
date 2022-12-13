---
{"dg-publish":true,"permalink":"/cpe/projet-sci/projet-scientifique/"}
---


#CPE 

## Objectifs
Réaliser d'une part un simulateur d'incendie permettant la création, le suivi et la propagation de feux de différents types (localisés sur une carte), et d'autre part de créer un dispositif de gestion de services d'urgences permettant, à partir d'informations collectées par des capteurs, de déployer et gérer les dispositifs adaptés pour éteindre les incendies. 


## TODO
- [ ] Liste tests (unitaires & E2E) IOT
- [x] Diagramme séquence
- [x] Protocole réseau
- [x] Structure de projet

## Jalons
- [x] TD: Présentation architecture globale et gestion du projet 📅 2022-12-08 ✅ 2022-12-08
- [ ] TD: Présentation conception logicielle. **Diagramme de classe / séquence** et Schéma BD 📅 2022-12-13
- [ ] TD: Présentation / démo chaîne IOT 📅 2023-01-04
	- [ ] Démo collecte, envoi et réception des données des feux dans la ville ainsi que génération d'appel REST.
- [ ] TD: Soutenance + démos finales 📅 2023-01-13

<!--
## Pourquoi du Rust en IOT est pertinent ?
- Support aisé de nouvelles plateformes si la target est supportée (no_std)
- Ref. RustConf 2022 contrôle des trains https://youtu.be/qaj5q88eLjk
- Un compilateur qui aide à éviter toutes les erreurs d'allocations / d'accès indexés
- Une création et implémentation de protocole simplifié grâce à [serde](https://lib.rs/serde) et [nom](https://lib.rs/nom)
- Documentation spécialisée pour le MicroBit ([MicroRust](https://droogmic.github.io/microrust) [Librairie microbit V1](https://lib.rs/crates/microbit))
- Librairie basée sur une [couche d'abstraction](https://github.com/nrf-rs/nrf-hal) pour les appareils NRF
- Compatible avec [l'écosystème d'outils Knurling](https://knurling.ferrous-systems.com/tools/) 
- Radio examples : [nrf-rs/microbit/pull/90](https://github.com/nrf-rs/microbit/pull/90/files)
-->
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
- Curve255 19 ? => Clé 128bits

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
	- Timestamp dernière MAJ
	- Charge du capteur

### Proto radio / UART
Objectifs : Mesh, résistance aux interférences, check is alive
- Les broadcast physiques ne sont pas repartagés
- Les broadcast logiques sont repartagés
- Soft max 125 de charge / device (avec saturation possible)


### Découpage trame
> https://lancaster-university.github.io/microbit-docs/ubit/radio/#capabilities
> Typically 32 bytes, but reconfigurable in code up to 1024 bytes.
> 4 trames stockées en RAM (16KB de RAM)

Here : [landcaster-university/microbit-dal/inc/drivers/MicroBitRadio.h#L68](https://github.com/lancaster-university/microbit-dal/blob/602153e9199c28c08fd2561ce7b43bf64e9e7394/inc/drivers/MicroBitRadio.h#L68)

### Trame statique
- **Une évolution possible serait de faire une trame de taille dynamique pour diminuer la saturation réseau**
- [ ] TD: Ajout `Request ID` => identifiant de la requête sur le système source

| Taille en bits | Nom                     | Justification                                                                               |
| -------------- | ----------------------- | ------------------------------------------------------------------------------------------- |
| 8              | Commande de trame       | 256 commandes différentes est largement suffisant                                           |
| 8              | Nombre de hop parcourus | 125 machines max passant par un microbit, donc aucune chance d'atteindre 256                |
| 32             | Adresse source          | Adresse physique de taille 32bits                                                           |
| 32             | Adresse destination     | Adresse physique de taille 32bits                                                           |
| 32             | Timestamp               | Unix timestamp en u32, viable jusqu'à 2106. D'ici la les microbit v1 ne fonctionneront plus |
| 8              | Partie de la requête    | 1 -> 255 (0 = Requête non partitionnée)                                                     |
| 8              | Nombre de parties       | 1 -> 255                                                                                    |
| 1912           | Donnée                  | $(255 * 8) - (8 * 4 + 32 * 3) = 1912$                                                                                             |

### Différentes requêtes

| Request ID | Request Abbrv | Connection           | Ident               | Description                                            | Champs                 |
| ---------- | ------------- | -------------------- | ------------------- | ------------------------------------------------------ | ---------------------- |
| 0          | `HELOP`       | Broadcast            | Physical            | Utilisé pour découvrir l'état initial du réseau local  |                        |
| 1          | `OLEHP`       | Targeted             | Logical => Physical | Partage initial de la table de machines                | Table des machines     |
| 2          | `IDENT`       | Targeted             | Physical => Logical | Demande au manager de mesh un identifiant de mesh      | @Physique              |
| 3          | `TNEDI`       | Targeted             | Logical => Physical | Assigne un identifiant de mesh                         | @Phy, @Mesh            |
| 4          | `TRUSTB`      | Broadcast (depth -1) | Logical             | Partage la clef publique du manager                    | pubkey                 |
| 5          | `TRUSTT`      | Targeted             | Logical => Logical  | Partage la clef publique de la machine                 | pubkey                 |
| 6          | `HELOL`       | Broadcast (depth 1)  | Logical             | Met a jour de manière forcée la table des machines     |                        |
| 7          | `OLEHL`       | Targeted             | Logical => Logical  | Partage de la table de machines si adresse est connue  | Table des machines     |
| 8          | `PUSH`        | Targeted             | Logical => Logical  | Envoie une mise à jour d'un capteur                    | Valeur/Type/Id capteur |
| 9          | `PUSH_ACK`    | Targeted             | Logical => Logical  | Confirme que les données ont correctement été envoyées | Capteur req ID         |
| 10         | `ADENY`       | Targeted             | Logical => Physical | Accès non autorisé                                     | Capteur req ID         |
| 11         | `UDENY`       | Targeted             | Logical => Logical  | Inconnu dans la table d'identifiants                   | Capteur req ID         |
| 12         | `ADD`         | Broadcast            | Logical             | Broadcast de la machine dans le réseau                 | Info machine           |
| 13         | `DEL`         | Broadcast            | Logical             | Perte d'une machine dans le réseau                     | Adresse machine        |

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

## Diagrammes séquence
### Dans le cadre de la simulation

#### Restitution données capteur
```seqdiag
seqdiag {
  "Collecteur" -> "Serveur relai" [label = "PFP(uart) cmd: PUSH\n(CID: 1\n,Type: Temp\n,Val: 1000)"];
  "Serveur relai" -> "Serveur relai" [label = "Interprétation de la mise à jour de donnée capteur"];
  "Serveur relai" -> "Broker Mosquito" [label = "Ajout d'une entrée\nvia le broker"];
  "Serveur relai" <-- "Broker Mosquito";
  "Collecteur" <-- "Serveur relai" [label = "PFP(uart) cmd: PUSH_ACK"];
}
```

#### Simulation d'un feu
```seqdiag
seqdiag {
  "Controlleur de simulation" -> "Serveur de simulation" [label = "JSON (HTTP)\n(Paramètres du feu)"];
  "Serveur de simulation" -> "Capteurs simulés" [label = "PFP(uart) cmd: PUSH"];
  "Capteurs simulés" -> "Collecteur" [label = "PFP(radio) cmd: PUSH\n(CID: 1\n,Type: Temp\n,Val: 1000)"];
  "Capteurs simulés" <-- "Collecteur" [label = "PFP(uart) cmd: PUSH_ACK"];
  "Serveur de simulation" <-- "Capteurs simulés";
  "Controlleur de simulation" <-- "Serveur de simulation";
}
```

### Dans le cadre du protocole (cas réel)
#### Connection d'un nouveau capteur
```seqdiag
seqdiag {
  browser  -> webserver [label = "GET /seqdiag/svg/base64"];
  webserver  -> processor [label = "Convert text to image"];
  webserver <-- processor;
  browser <-- webserver;
}
```

#### Perte d'un capteur
```seqdiag
seqdiag {
  "Capteur X" -> "Capteur X" [label = "Vérification des TTL"];
  "Capteur X" -> "Capteur P" [label = "PFP(radio) cmd: DEL\n(Id logique capteur L)"];
  "Capteur X" <-- "Capteur P";
  "Capteur X" -> "Collecteur" [label = "PFP(radio) cmd: DEL\n(Id logique capteur L)"];
  "Capteur X" <-- "Collecteur";
  "Capteur P" -> "Collecteur" [label = "PFP(radio) cmd: DEL\n(Id logique capteur L)"];
  "Capteur P" <-- "Collecteur";
}
```

#### Création d'un mesh
```seqdiag
seqdiag {
  browser  -> webserver [label = "GET /seqdiag/svg/base64"];
  webserver  -> processor [label = "Convert text to image"];
  webserver <-- processor;
  browser <-- webserver;
}
```

#### Perte du relai
```seqdiag
seqdiag {
  browser  -> webserver [label = "GET /seqdiag/svg/base64"];
  webserver  -> processor [label = "Convert text to image"];
  webserver <-- processor;
  browser <-- webserver;
}
```