# Protocole radio

Notes du 19 octobre 2026. Retour au [sommaire](../README.md). Circuit : [materiel.md](materiel.md).

## Principe

L’application ne se connecte pas au ventilateur. Il n’y a pas de service GATT à écrire. Le téléphone, comme la télécommande, **diffuse une trame publicitaire BLE** (non connectable). Le récepteur dans le pavillon du ventilateur l’exécute. Pas d’appairage Bluetooth au sens du téléphone.

L’application qui va avec cette famille s’appelle **FanLamp Pro**. Le codec est dans sa bibliothèque native (`libencodeV3.so` : `ble_v2_encode`, `ble_v3_encoder`, `AES_ECB_encrypt`). L’application émet souvent deux variantes, v2 et v3. La télécommande de ce ventilateur parle la variante **v2**, type d’appareil `0x0100`.

Sur la trame capturée, l’octet de signature vaut 0 : le corps n’est pas un bloc AES complet. Le déballage utilisé est celui de ha-ble-adv, FanLamp V2 (blanchiment + CRC16).

C’est maintenu ici :

- intégration Home Assistant : [NicoIIT/ha-ble-adv](https://github.com/NicoIIT/ha-ble-adv)
- relais ESP32 : [NicoIIT/esphome-ble_adv_proxy](https://github.com/NicoIIT/esphome-ble_adv_proxy)
- récit d’un clonage de télécommande FanLamp : [Wesley Ray, juin 2026](https://garden.c0smere.net/Campsite-Automation/Gazebo-Fans-%E2%80%94-Cloning-the-FanLamp-Remote)

Le ventilateur sort d’usine déjà lié à **cette** télécommande. Inventer un nouvel identifiant et couper le courant ne suffit pas toujours. La voie qui a marché est de **copier l’identifiant déjà émis**.

Dans l’intégration, ce codec s’appelle `remote_v21` (`lampsmart_pro_v2/r`, paramètres `[False, [0x10, 0x00, 0x56]]`). Le jumeau `remote_v31` a le même préfixe avec le drapeau de signature à vrai. Il n’échange pas le marche et l’arrêt de la lumière.

Les traducteurs de lumière de ce codec : ON = `0x10`, OFF = `0x11`. Le menu **Jeu de traducteurs** propose seulement Par défaut, Pas de mode Nuit, Mode Nuit comme Effet, Mode Nuit comme seconde lampe. « Inverser ON/OFF » existe sur d’autres codecs, pas sur celui-ci. « Inverser Chaud / Froid » ne concerne que la température de couleur.

## Capture nRF Connect — 19 octobre 2026, 14 h 57

La télécommande n’émet que pendant qu’une touche est maintenue. Dans une longue liste, elle se reconnaît au signal (ici −21 dBm contre le téléphone, −41 dBm un peu plus loin) et au Service Data `08F0`.

| Champ nRF Connect | Valeur |
|---|---|
| Connectable | non |
| Signal | −21 dBm télécommande contre le téléphone |
| Cadence | un paquet toutes les 8 à 45 ms, plus de 2 000 paquets tant que la touche reste enfoncée |
| Service Data | UUID `08F0`, en-tête `F0 08` |

Exemple copié, touche **1**, 24 octets après l’UUID :

```
10 00 4A F3 D1 C1 8E F3 07 DC 52 F0 5E C3 F6 45 26 2F E3 76 C4 D1 7E 5F
```

Le contrôle CRC16 de cette trame est valide. Une fois déballée :

| Champ | Valeur |
|---|---|
| Famille | type d’appareil `0x0100`, variante v2, en-tête `F0 08` |
| Préfixe décodé | `[0x10, 0x00, 0x56]` |
| Identifiant de la télécommande | `0x148D` (5261 en décimal) |
| Index | `0` |
| Compteur | `6` sur cet exemplaire, il augmente à chaque paquet |
| Commande | `0x31`, argument `0x20`, second argument `1` |
| Signature | absente |

C’est la commande **ventilateur allumé, vitesse 1**. Le bloc hexadécimal change à chaque paquet : seul l’identifiant `0x148D` est stable. Rejouer dans nRF Connect « Clone as Advertiser » renvoie un paquet déjà vu, le ventilateur l’ignore. Home Assistant doit émettre un compteur neuf.

Sur iOS, les publicités non connectables sont souvent masquées. Elles étaient visibles ici, donc le téléphone ne les bloquait pas.

## Pourquoi l’application Apple peut échouer

Le QR de l’étiquette dit « Scan to get support and user manual ». La photo est trop floue pour en sortir l’URL. Ce n’est pas forcément le lien App Store.

Pistes concrètes, sans ordre :

- le QR ouvre une page de notice ou un APK, pas l’App Store ;
- FanLamp Pro demande un compte et le Bluetooth, et la fenêtre d’appairage dure moins de 5 secondes après le retour du courant ;
- iOS révoque le Bluetooth si l’autorisation n’a pas été donnée à l’installation.

L’intégration Home Assistant court-circuite l’application. La marche à suivre est dans [home-assistant.md](home-assistant.md).
