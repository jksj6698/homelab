# Mises à jour

[← Sommaire](../README.md)

Les mises à jour sont automatiques, avec deux règles : **les correctifs de sécurité passent au plus vite**, et **le
reste avance par anneaux**, des machines les moins critiques aux plus exposées. Un problème apparaît ainsi d'abord là
où il coûte le moins.

## Les anneaux

```mermaid
flowchart LR
    A1["<b>Anneau 1</b> · chaque nuit<br/>machines internes peu critiques<br/>build, supervision, aperçus internes"]
    A2["<b>Anneau 2</b> · deux fois par semaine<br/>services internes<br/>dépôt local, signature, mail, forge"]
    A3["<b>Anneau 3</b> · une fois par semaine<br/>machines exposées et accès<br/>reverse proxy, serveurs web, bastion"]
    A1 -- "les mêmes paquets,<br/>quelques jours plus tard" --> A2 -- "puis" --> A3
```

## Le kit de mises à jour

Les machines reçoivent toutes le même kit (un script et quatre minuteurs systemd), installé depuis le dépôt local
après vérification de l'empreinte de chacun de ses fichiers.

| Passage | Rythme | Rôle |
|---|---|---|
| sécurité | plusieurs fois par jour | correctifs des origines de sécurité seulement ; ne redémarre rien |
| anneau | la nuit, les jours de l'anneau | toutes les mises à jour, en gardant les configurations locales |
| nuit | chaque nuit, un anneau après l'autre | redémarre la machine ou les services qui en ont besoin, si c'est sans risque |
| métriques | en continu | publie l'état de la machine pour la supervision |

Le passage de nuit ne redémarre jamais une machine à l'aveugle :

```mermaid
flowchart TB
    S["passage de nuit"] --> P{"frein à main<br/>central posé ?"}
    P -- oui --> X["rien"]
    P -- non --> N{"redémarrage<br/>nécessaire ?"}
    N -- non --> Z["rien"]
    N -- oui --> T{"la machine restera-t-elle<br/>joignable par Teleport ?"}
    T -- non --> AL["annule et alerte"]
    T -- oui --> K{"nouveau noyau<br/>en attente ?"}
    K -- oui --> R["redémarre la machine"]
    K -- non --> SV["relance seulement<br/>les services touchés"]
```

- **Vérification Teleport** : le service démarre avec la machine, le nom du bastion se résout à la bonne adresse, et
  cloud-init ne réécrira pas le fichier `hosts` au démarrage. Ce sont trois causes réelles de machines devenues
  injoignables après un redémarrage.
- **Frein à main** : un interrupteur central suspend tous les passages, ou un seul anneau.
- **Verrou commun** : deux passages ne tournent jamais en même temps sur une machine.
- **Agents de sécurité mis à jour à part** : Teleport, Wazuh et CrowdSec suivent leur propre rythme ; un agent Wazuh
  ne doit jamais dépasser la version de son serveur.
- **Raspberry Pi et ZFS** : le module ZFS est recompilé pour chaque nouveau noyau ; le redémarrage attend la fin de la
  compilation.

## Images de conteneurs

- **Un catalogue des versions autorisées**, tenu à jour automatiquement : il n'accepte qu'un correctif de la même
  branche, jamais une préversion ni un changement de version majeure. Les nouvelles branches majeures sont signalées
  une fois par mois, pour décision.
- **Images maison reconstruites sur place**, à partir de sources vérifiées (Traefik, nginx, pare-feu web).
- **Côté machines**, au rythme de leur anneau : la nouvelle version est appliquée, les réponses des services sont
  comparées avant et après, et **la version précédente revient seule** si quelque chose ne répond plus. Une version qui
  a échoué est bloquée sur cette machine et signalée.

## Exemple : un service installé nativement

Pour un service distribué seulement en image Docker mais installé sans Docker, le dépôt local extrait le programme de
l'image officielle ; le service se met ensuite à jour lui-même, avec retour arrière :

```mermaid
flowchart LR
    REL["Dépôt local<br/>extrait le programme<br/>de l'image officielle,<br/>publie version et empreinte"]
    Q{"nouvelle version ?"}
    RIEN["rien à faire"]
    INS["copie la base,<br/>installe à côté de l'ancienne,<br/>bascule, redémarre"]
    OK{"test de santé<br/>dans la minute ?"}
    BON["nouvelle version<br/>en service"]
    RB["retour à l'ancienne version<br/>et à la copie de la base"]
    REL --> Q
    Q -- non --> RIEN
    Q -- oui --> INS --> OK
    OK -- oui --> BON
    OK -- non --> RB
```

Les dernières versions et les dernières copies de la base restent sur place, pour revenir en arrière même plus tard.
