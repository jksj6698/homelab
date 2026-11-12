# Retours d'expérience

[← Sommaire](../README.md)

Des pannes et des pièges rencontrés pour de vrai, tous corrigés. Chacun a laissé une règle.

## Réseau et système

**Une bascule réseau à distance, sauvée par son retour arrière.** En passant un Raspberry Pi sur un pont réseau pour
Proxmox, NetworkManager, relancé par un service annexe, a repris la carte réseau : plus d'accès. Le service de retour
arrière, armé avant la bascule, a restauré l'ancienne configuration au bout de quelques minutes sans confirmation.
*Règle : toute modification réseau ou pare-feu à distance part avec un retour arrière automatique.*

**Un Raspberry Pi 4 n'a pas d'horloge matérielle.** Sans réseau au démarrage, il repart à la date de son image, et APT
refuse des signatures « pas encore valides ». *Règle : enregistrer l'heure à l'arrêt (`fake-hwclock`) et synchroniser
sur une source interne.*

**cloud-init réécrivait le fichier `hosts` à chaque redémarrage**, effaçant les entrées dont dépendait l'agent
Teleport : des machines disparaissaient du bastion après un redémarrage. L'option était imposée par les données
générées par l'hyperviseur, prioritaires sur la configuration locale. *Règle : `hosts` figé et cloud-init désactivé une
fois la machine installée ; et le kit de mises à jour vérifie ce point avant tout redémarrage.*

**Un conteneur Debian 13 sur PXVirt démarrait sans adresse IPv4.** L'hyperviseur écrivait la configuration pour
ifupdown, alors que l'image utilise systemd-networkd. *Règle : un fichier `.network` statique dans le conteneur.*

## Conteneurs

**Un profil seccomp partagé a cassé un formulaire de contact.** Prévu pour nginx, il a été réutilisé pour un binaire
Go, qui avait besoin de `getpeername` : plus aucun mail ne partait. *Règle : un profil seccomp par famille de runtime.*

**Une adresse de conteneur qui change inverse une règle de pare-feu en silence.** Au redémarrage, Docker a attribué
les adresses dans un autre ordre : le filtrage sortant visait le mauvais conteneur. *Règle : adresse fixe pour tout
conteneur dont une règle dépend.*

**`tmpfs` sans `mode=1777` : nginx non root en boucle de redémarrage.** Docker, contrairement à Podman, ne recopie pas
le contenu de l'image dans un `tmpfs` monté par-dessus. *Règle : mode explicite, et un test de redémarrage réel avant
de déclarer un service prêt.*

**`flush ruleset` en tête de la configuration nftables effaçait les chaînes de Docker** à chaque rechargement du
pare-feu. *Règle : ne remplacer que sa propre table, jamais tout le jeu de règles.*

## Chaîne d'approvisionnement et certificats

**Un cache APT a rangé des paquets sous le nom d'autres paquets.** Des demandes envoyées à la chaîne vers un miroir
qui répondait par des redirections ont mélangé les réponses. Les signatures APT ont refusé les fichiers : rien de faux
n'a été installé. *Règle : vérifier les empreintes partout, et envoyer les demandes une par une vers ce miroir.*

**Let's Encrypt refuse une clé Ed25519 pour un certificat serveur**, même après un défi DNS réussi. *Règle : EC P-256
pour le certificat, Ed25519 seulement pour le transport SSH.*

**Un faux échec de vérification TLS.** `openssl s_client` sans `-CAfile` affiche une erreur de chaîne alors que le
certificat est valide. *Règle : prouver la cause avant de corriger quoi que ce soit.*

## Détection et accès

**Un fichier-piège qui sonne pour rien.** Le premier matin, un leurre a déclenché une alerte d'intrusion. Le journal
d'audit a tout de suite nommé le coupable : le scanner d'intégrité, qui lit chaque fichier pour en calculer l'empreinte
— les leurres compris — sous aucun compte humain. Vraie lecture, mais lecteur légitime. *Règle : un piège doit ignorer
ses lecteurs légitimes connus (le scanner d'intégrité) et ne compter que les lectures qui n'ont rien à y faire.*

**Donner un rôle de plus à l'agent d'accès de la machine par laquelle on y est connecté.** Ajouter la publication d'une
console web à l'agent du bastion, sur la machine même qu'on administre à travers lui : un redémarrage raté de l'agent,
et on perd l'accès. Le changement a été posé par un garde indépendant de la session, qui redémarre l'agent, vérifie
qu'il revient sain, et restaure l'ancienne configuration tout seul sinon. Deux pièges au passage : un en-tête transmis
réservé, que l'agent pose lui-même et refuse qu'on réécrive ; et un rôle nouveau qui ne s'obtient pas en silence, mais
en re-rejoignant le bastion avec un jeton d'enrôlement neuf. *Règle : une modification du chemin d'accès embarque son
propre garde et son retour arrière, détachés de la session en cours.*
