# didactic-sniffle

Ce dépôt documente le protocole de communication entre le contrôleur principal
Raspberry Pi et le périphérique esclave embarqué. Le protocole utilise I²C avec
une ligne d’interruption active à l’état bas pour les événements asynchrones.

Consultez [`docs/communication-protocol.md`](docs/communication-protocol.md)
pour une spécification détaillée qui décrit :

- Le contexte système et les caractéristiques du transport.
- La structure des trames (longueur, identifiants d’action/événement, charge
  utile, CRC).
- Les politiques de gestion des erreurs, de relance et de surveillance
  (watchdog).
- Les recommandations de test et de validation afin que firmwares, pilotes et
  outils de diagnostic restent interopérables.
