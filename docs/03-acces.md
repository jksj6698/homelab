# Accès d'administration

[← Sommaire](../README.md)

**Les serveurs s'administrent par Teleport**, joignable seulement à travers le VPN du pare-feu : en dehors du VPN,
rien de ce qui sert à administrer l'infra n'est joignable depuis internet.

Teleport (édition Community) a été préféré à JumpServer : un seul binaire, et des certificats de courte durée liés à
l'identité plutôt qu'un coffre de secrets à héberger, là où JumpServer demande 4 cœurs, 8 Go de RAM et 100 Go de
disque au minimum.

```mermaid
flowchart LR
    ADM(["Administrateur"])
    VPN{{"VPN WireGuard<br/>du pare-feu"}}
    subgraph TP["Bastion Teleport"]
        AUTH["Auth<br/>autorité de certificats,<br/>comptes, audit"]
        PROXY["Proxy<br/>terminal, interfaces web"]
    end
    NODES["Serveurs<br/>VM et conteneurs"]
    APPS["Interfaces web<br/>d'administration"]

    ADM --> VPN --> PROXY
    PROXY --- AUTH
    NODES -- "tunnel sortant" --> PROXY
    PROXY -- "HTTPS" --> APPS
```

## Comment se passe une connexion

```mermaid
sequenceDiagram
    actor A as Administrateur
    participant P as Proxy Teleport
    participant U as Auth Teleport
    participant N as Serveur
    Note over N,P: au démarrage, le serveur ouvre un tunnel sortant vers le proxy
    A->>P: connexion par le VPN, mot de passe et second facteur
    P->>U: vérifie l'identité
    U-->>A: certificat SSH de courte durée
    A->>P: tsh ssh serveur
    P->>N: session relayée dans le tunnel
    N-->>U: session enregistrée et journalisée
```

- **L'administration n'ouvre aucun port entrant** : chaque serveur ouvre lui-même un tunnel sortant vers le proxy, en
  épinglant l'autorité de certificats de Teleport. Chaque serveur reçoit son propre jeton d'enrôlement, supprimé dès que
  le serveur apparaît.
- **Pas de clé SSH personnelle à distribuer ni à révoquer** : l'accès par Teleport repose sur des certificats de courte
  durée, délivrés après double authentification.
- **Sessions enregistrées** et journalisées.
- **Interfaces web** : chaque interface d'administration a sa propre adresse derrière Teleport, couverte par un
  certificat joker ; les nouvelles se déclarent dynamiquement, sans redémarrer le bastion.
- **Les mises à jour de nuit vérifient, avant tout redémarrage, que la machine restera joignable par Teleport** :
  service activé au démarrage, nom du bastion bien résolu, fichier `hosts` protégé contre cloud-init. Sinon, le
  redémarrage est annulé et une alerte part (voir [mises à jour](09-mises-a-jour.md)).
