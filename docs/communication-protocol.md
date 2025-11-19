# Spécification du protocole de communication

## Contexte système

Le système comprend un Raspberry Pi (RPi) agissant comme contrôleur maître et
un microcontrôleur embarqué agissant comme périphérique esclave. Le RPi pilote
la logique applicative haut niveau, initie toutes les transactions I²C et expose
les interfaces utilisateur (CLI, UI ou cloud). L’esclave embarqué exécute les
actions matérielles sensibles au temps, collecte la télémétrie et publie des
événements asynchrones lorsque l’état local change.

### Répartition des rôles
- **Maître (RPi)** : configure le bus, cadence les transactions jusqu’à 400 kHz
  et envoie les commandes vers l’esclave. Il traite aussi les interruptions qui
  signalent des événements côté esclave, planifie les lectures de suivi et
  applique un filtrage pour éviter une surcharge de l’hôte.
- **Esclave (cible embarquée)** : expose une fenêtre de registres que le maître
  interroge via I²C, stocke les trames d’événements à destination du maître,
  valide les trames entrantes et publie l’état du watchdog/santé.

## Caractéristiques du transport I²C

| Paramètre | Valeur |
| --- | --- |
| Vitesse de bus | 100 kHz (mode standard) nominal, 400 kHz en mode rapide si les deux appareils le supportent |
| Adressage | Adresse esclave 7 bits fixe `0x2A` |
| Résistances de tirage | 4,7 kΩ sur SDA/SCL au niveau de la carte porteuse |
| Domaine électrique | Logique 3,3 V ; tous les appareils doivent être tolérants |
| Ligne d’interruption | Collecteur ouvert, active à l’état bas, tirée vers 3,3 V |

### Utilisation de la ligne d’interruption
- L’esclave force la ligne à l’état bas lorsqu’il place une ou plusieurs trames
  d’événement dans sa file ou lorsqu’un changement d’état urgent nécessite une
  attention immédiate du maître.
- Le maître acquitte l’interruption en lisant la file d’événements. L’esclave ne
  relâche la ligne qu’une fois la file sortante vide.
- Les impulsions parasites plus courtes que 20 µs doivent être ignorées par le
  maître ; le firmware esclave applique un étirement minimal de 50 µs pour
  garantir la détection.

## Format de trame

Toutes les trames partagent la même structure, quel que soit le sens. Une trame
correspond à une transaction I²C. Le maître écrit toujours l’octet de longueur
avant les données lorsqu’il envoie une trame, et lit cet octet en premier lorsqu’il
reçoit une trame.

```
+---------+-----------------+--------------------+--------------+
| Octet # | Champ           | Description        | Notes        |
+=========+=================+====================+==============+
| 0       | Longueur        | Nombre d’octets    | Inclut ID,   |
|         |                 | suivants (1–62)    | paramètres,  |
|         |                 |                    | CRC.         |
+---------+-----------------+--------------------+--------------+
| 1       | ID action/évén. | Opcode de commande | `0x00`..`0x7F`
|         |                 | ou identifiant     | pour actions,|
|         |                 | d’événement        | `0x80`..`0xFF`
|         |                 |                    | pour événements. |
+---------+-----------------+--------------------+--------------+
| 2..N-2  | Paramètres      | Octets de charge   | Optionnels.  |
|         |                 | utile              |              |
+---------+-----------------+--------------------+--------------+
| N-1     | CRC8            | CRC8 Dallas/Maxim  | Polynôme     |
|         |                 | sur Longueur..Par. | `x^8+x^5+x^4+1`. |
+---------+-----------------+--------------------+--------------+
```

- **Contraintes sur l’octet de longueur** : la taille minimale est `0x02` (ID +
  CRC). La valeur maximale `0x3E` (62 octets) garantit qu’une transaction tient
  dans un seul transfert I²C sans étirement d’horloge supérieur à 1 ms.
- **Charge utile paramètre** : jusqu’à 60 octets. Les entiers multi-octets sont
  codés en big-endian. Les indicateurs sont packés LSB en premier sauf
  spécification contraire.
- **CRC** : calculé sur l’octet de longueur et tous les octets suivants, à
  l’exception du champ CRC lui-même. Les deux appareils doivent rejeter toute
  trame dont le CRC ne correspond pas.

### Exemple de trame d’action
```
Octet 0 (Longueur)        : 0x05
Octet 1 (ID d’action)     : 0x12 (SET_PWM)
Octet 2 (Paramètre)       : 0x01 (canal)
Octets 3..4 (Paramètre)   : 0x03E8 (1000 counts, big-endian)
Octet 5 (CRC8)            : 0xA9
```

### Exemple de trame d’événement
```
Octet 0 (Longueur)        : 0x04
Octet 1 (ID d’événement)  : 0x82 (THERMAL_ALERT)
Octet 2 (Paramètre)       : 0x5A (température mesurée en °C)
Octet 3 (CRC8)            : 0xD4
```

## Politiques de gestion des erreurs

### Erreurs de CRC
- **Le maître reçoit un CRC invalide** : il jette la trame, journalise l’échec et
  demande une nouvelle lecture de la trame jusqu’à trois tentatives
  supplémentaires. Après trois échecs consécutifs, le maître remonte un
  avertissement à l’utilisateur et marque l’esclave comme dégradé jusqu’à
  réception d’une trame valide.
- **L’esclave reçoit un CRC invalide** : il ignore silencieusement la trame,
  incrémente un compteur d’erreurs exposé via un registre de diagnostic et étire
  SDA à l’état bas pendant un temps équivalent à un octet pour forcer le maître à
  détecter le NACK. Le maître doit retenter la commande une fois ; des échecs
  répétés entraînent une erreur applicative.

### Relances vs abandon
- **Commandes idempotentes** (lectures, requêtes de télémétrie) : jusqu’à trois
  relances avant de signaler une erreur à l’interface utilisateur.
- **Commandes non idempotentes** (actions modifiant l’état) : une seule
  relance. Si la relance échoue, le maître abandonne l’opération, marque le
  sous-système concerné comme indéterminé et demande une intervention manuelle.

### Watchdog esclave et remise en état du bus
- L’esclave exécute un watchdog de 25 ms qui réinitialise son périphérique I²C
  si SDA reste à l’état bas ou si les transactions se figent. Lorsque le watchdog
  se déclenche, le firmware passe SDA/SCL en haute impédance, vide les FIFOs
  internes et désactive la ligne d’interruption.
- Après récupération, l’esclave publie l’événement `0x8F (RECOVERED_RESET)` pour
  que le maître puisse recharger la configuration. Le maître doit relire l’état
  critique avant d’émettre de nouvelles commandes.

## Conseils de test et validation

### Séquences commande/réponse
1. Envoyer la commande `PING (0x01)` à chaque démarrage et vérifier que
   l’esclave renvoie la trame attendue.
2. Exécuter chaque action dans l’ordre croissant des opcodes en vérifiant que
   chaque réponse correspond à la charge utile et au CRC documentés.

### Flux de notification d’événements
- Déclencher chaque événement matériel (thermique, alimentation, watchdog) et
  confirmer :
  1. L’esclave force la ligne d’interruption.
  2. Le maître lit la file en moins de 10 ms.
  3. La file se vide et la ligne d’interruption est relâchée.

### EMI / Injection de défauts
- Introduire un étirement d’horloge artificiel ou perturber SDA en cours de
  trame pour provoquer des erreurs de CRC. Vérifier que les compteurs de relance
  augmentent et que les avertissements utilisateur n’apparaissent qu’après
  épuisement du budget de relance configuré.
- Court-circuiter manuellement SDA à la masse pendant 50 ms pour déclencher le
  watchdog esclave. Vérifier que le maître observe l’événement
  `RECOVERED_RESET` et recharge la configuration sans redémarrer l’hôte.

### Liste de contrôle de régression
- Les tests unitaires automatisés doivent inclure un ensemble encodeur/décodeur
  de trames couvrant le polynôme de CRC et les tailles limites (2 octets,
  62 octets).
- Les tests Hardware-in-the-loop (HIL) doivent tourner quotidiennement pour
  valider la latence d’interruption et capturer les formes d’onde du bus pour
  l’analyse post-mortem.
