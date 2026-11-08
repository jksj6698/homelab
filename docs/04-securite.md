# Sécurité

[← Sommaire](../README.md)

Défense en profondeur : chaque couche suppose que la précédente peut céder.

```mermaid
flowchart LR
    A(("Internet"))
    L1["<b>1 · Périmètre</b><br/>pare-feu central<br/>NAT vers le reverse proxy seulement<br/>bannissements CrowdSec"]
    L2["<b>2 · DMZ</b><br/>reverse proxy, TLS<br/>WAF Coraza, OWASP CRS<br/>correctifs virtuels"]
    L3["<b>3 · Réseau interne</b><br/>zones étanches, refus par défaut<br/>DNS cloisonné<br/>pas d'internet direct par défaut"]
    L4["<b>4 · Hôtes</b><br/>administration par Teleport<br/>agents Wazuh et CrowdSec<br/>pare-feu local sur les machines sensibles<br/>mises à jour automatiques"]
    L5["<b>5 · Applications</b><br/>non root, sans capacité<br/>système en lecture seule<br/>seccomp par runtime"]
    A --> L1 --> L2 --> L3 --> L4 --> L5
```

La détection tourne en boucle au-dessus de ces couches :

```mermaid
flowchart LR
    subgraph M["Sur les machines"]
        AW["agent Wazuh"]
        AC["agent CrowdSec"]
        NE["node_exporter"]
    end
    WZ["SIEM Wazuh<br/>intégrité, vulnérabilités, alertes"]
    LAPI["Pare-feu central<br/>API CrowdSec"]
    BO{{"bouncer<br/>du pare-feu"}}
    PR["Supervision<br/>Prometheus, Alertmanager"]
    MAIL(["Alertes par mail<br/>via le relais"])
    AW --> WZ
    AC -- "signalements" --> LAPI -- "décisions" --> BO
    NE --> PR --> MAIL
    WZ --> MAIL
```

## Périmètre et DMZ

- **Une seule machine reçoit le trafic des sites** : le reverse proxy (Traefik), avec un pare-feu web Coraza et l'OWASP
  Core Rule Set en sidecar. Les serveurs web derrière lui ne sont jamais exposés directement.
- **Correctifs virtuels** : un fichier de règles maison, chargé à part, permet de bloquer une attaque connue sans
  reconstruire l'image.
- **CrowdSec** : les machines ne font que détecter ; l'API et le bouncer sont sur le pare-feu central, qui bloque
  l'adresse pour toute l'infra.

## Hôtes

- **Administration par Teleport**, avec double authentification (voir [accès](03-acces.md)).
- **Pare-feu local** sur les machines sensibles, en plus du pare-feu central : par exemple, le relais de fichiers du
  dépôt local n'accepte qu'une liste d'adresses, et l'autorité de signature n'accepte que les machines autorisées à
  faire signer.
- **Wazuh** : des agents sur les machines, surveillance d'intégrité en temps réel des fichiers sensibles, détection
  des vulnérabilités, alertes par mail.
- **Comptes de service à commande forcée** : sauvegardes, déploiements et signatures de certificats passent par des
  comptes dédiés dont la clé SSH ne peut lancer qu'un seul script (`restrict`, `from=`, `command=`), avec `sudo` limité
  à ce script.

## Conteneurs et services

Les images maison appliquent par défaut :

- utilisateur non root dès le démarrage ;
- `cap_drop: ALL`, et seulement le strict nécessaire en retour (par exemple `NET_BIND_SERVICE`) ;
- système de fichiers en lecture seule, `tmpfs` en `mode=1777` pour ce qui doit être écrit ;
- **un profil seccomp propre à chaque famille de runtime** : un profil prévu pour nginx, réutilisé pour un binaire Go,
  bloquait `getpeername` et cassait l'envoi de mails ;
- limites de mémoire et de processus ;
- **adresse IP fixe** dès qu'une règle de pare-feu en dépend ;
- images `FROM scratch` ou nginx statique quand c'est possible.

Les services installés nativement suivent la même idée avec systemd : compte dédié, une seule capacité si besoin,
`NoNewPrivileges`, système en lecture seule sauf les données (`ProtectSystem=strict`), `/tmp` privé.

## Chaîne d'approvisionnement

- **Tout passe par le dépôt local** : paquets APT, fichiers, images de conteneurs (voir
  [approvisionnement](05-approvisionnement.md)).
- **Empreintes vérifiées** pour tout ce qui est téléchargé hors APT ; APT vérifie déjà ses signatures, ce qui a permis
  de repérer un cache corrompu sans rien installer de faux.
- **Releases tierces vérifiées** contre la somme publiée par l'éditeur, jamais choisies sur leur seul nom de fichier.

## Secrets

- **Un détenteur par secret** : les jetons de l'API DNS ne sont que sur l'autorité de signature, les identifiants
  d'envoi de mail que sur le relais.
- **Jamais de secret en argument de commande ni dans un journal** : saisie masquée, passage par l'entrée standard.
- **Les clés privées des certificats ne quittent jamais leur machine** : seule la demande de signature circule (voir
  [certificats](08-certificats.md)).
