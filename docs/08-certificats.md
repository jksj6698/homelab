# Certificats

[← Sommaire](../README.md)

Les interfaces internes utilisent de vrais certificats Let's Encrypt, obtenus par défi DNS : aucune machine interne n'a
besoin d'être joignable depuis internet. **Une seule machine détient les jetons de l'API DNS**, l'autorité de
signature. Les autres lui envoient une demande de signature ; **la clé privée d'un certificat ne quitte jamais sa
machine**.

## Le relais de signature

```mermaid
sequenceDiagram
    participant H as Machine
    participant C as Autorité de signature
    participant L as Let's Encrypt
    participant O as API DNS du domaine
    H->>H: génère sa clé (EC P-256) et sa demande de signature
    H->>C: SSH, demande sur l'entrée standard
    Note over C: clé SSH limitée : adresse de la machine,<br/>commande forcée avec le seul nom autorisé
    C->>C: vérifie que la demande ne porte que ce nom
    C->>L: commande ACME
    L-->>C: défi DNS-01
    C->>O: publie l'enregistrement TXT
    L-->>C: certificat signé
    C-->>H: certificat et chaîne sur la sortie standard
    H->>H: installe, relance le service
```

- **Une clé SSH par machine** sur l'autorité de signature, avec `restrict`, `from=` (l'adresse de la machine) et
  `command=` (le script de signature et le nom exact autorisé) : une machine ne peut faire signer que son propre nom.
- **Pare-feu local** : l'autorité de signature n'accepte que les machines autorisées à faire signer.
- **Clé du certificat en EC P-256**, jamais Ed25519 : Let's Encrypt la refuse pour un certificat serveur. La clé SSH du
  transport, elle, peut rester en Ed25519 : ce sont deux clés sans rapport.
- **Sites publics** : le reverse proxy obtient et renouvelle lui-même leurs certificats, par défi HTTP.

## Renouvellement

Le renouvellement repasse par le même chemin, piloté par un minuteur sur la machine :

```mermaid
flowchart LR
    T["minuteur hebdomadaire<br/>sur la machine"] --> Q{"expire dans<br/>moins de 30 jours ?"}
    Q -- non --> F["rien à faire"]
    Q -- oui --> K["nouvelle clé<br/>et demande de signature"]
    K --> S["SSH vers<br/>l'autorité de signature"]
    S --> V{"chaîne reçue<br/>et valide ?"}
    V -- oui --> I["installe,<br/>relance le service"]
    V -- non --> A["garde le certificat en place,<br/>nouvel essai au passage suivant"]
```

Chaque renouvellement produit une nouvelle clé : une clé compromise ne survit pas au certificat suivant.
