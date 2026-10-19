# Ventilateur de plafond ZMISHIBO, piloté depuis Home Assistant

La télécommande de ce ventilateur ZMISHIBO 132 cm parle en **Bluetooth Low Energy**. L’application du fabricant, FanLamp Pro, n’a pas servi. Home Assistant reprend l’identifiant déjà gravé dans la télécommande et envoie les commandes par une carte ESP32-C3.

La télécommande d’origine reste appairée. Le courant du ventilateur n’a pas été coupé pour un nouvel appairage.

![Circuit de la télécommande, antenne en haut](docs/photos/01-pcb-face-composants.jpg)

## Résultat

| Fonction | Dans Home Assistant |
|---|---|
| Lampe, marche et arrêt | `light.lumiere_ventilateur` (bouton corrigé) |
| Luminosité et blanc chaud / froid | `light.ventilateur_salon_lampe_principale` |
| Ventilateur, 6 vitesses, sens, vent naturel | entités créées par l’intégration |
| Relais radio | ESP32-C3 `relais-ventilateur`, voyant sur GPIO8 |

Le pont Maison (Apple) existait déjà. La lumière corrigée n’y apparaît qu’après l’avoir cochée dans HomeKit Bridge.

## Pourquoi l’application ne sert pas

Ce n’est ni du 433 MHz ni de l’infrarouge. La fiche Amazon qui dit « infrarouge » est fausse pour cette télécommande : le cuivre en haut de carte est une antenne 2,4 GHz, il n’y a pas de LED infrarouge.

La puce unique (boîtier SOP-16, sans marquage lisible) diffuse des **publicités BLE non connectables**. Il n’y a pas de connexion GATT. Le téléphone, comme la télécommande, émet une trame déjà préparée. Le récepteur dans le pavillon la reconnaît et exécute.

Chaque paquet porte un compteur qui augmente. Rejouer une vieille trame (nRF Connect, « Clone as Advertiser ») ne fait rien : le ventilateur ignore un compteur déjà vu.

Détail du circuit, des photos et des schémas : [docs/materiel.md](docs/materiel.md).  
Détail du protocole et de la capture : [docs/protocole.md](docs/protocole.md).

## Matériel de cette installation

| | |
|---|---|
| Ventilateur | ZMISHIBO 132 cm, LED 20 W dimmable, moteur DC réversible, blanc/bois, classe F |
| Modèle sur la télécommande | **ZMCF008** |
| Piles télécommande | 2 × AAA |
| Identifiant radio copié | `0x148D`, codec `remote_v21` |
| Carte relais | TENSTAR ROBOT Super Mini, **ESP32-C3**, flash 4 Mo |
| Réseau | `masia-antoine.fr`, **2,4 GHz** (le C3 n’a pas le 5 GHz) |
| Logiciel carte | ESPHome **2026.9.1** |
| Intégration | [NicoIIT/ha-ble-adv](https://github.com/NicoIIT/ha-ble-adv), titre **BLE ADV Ceiling Fan / Lamps** |

## Reproduire

Home Assistant doit pouvoir **émettre** ces publicités. Le Bluetooth du téléphone ne compte pas. Ici, l’hôte Home Assistant n’avait aucun adaptateur capable de le faire. La carte ESP32-C3 sert de relais, via [esphome-ble_adv_proxy](https://github.com/NicoIIT/esphome-ble_adv_proxy). Un proxy Bluetooth classique (`bluetooth_proxy`) ne suffit pas.

1. **Repérer la télécommande** avec nRF Connect. Elle n’émet que tant qu’une touche est maintenue. Chercher une publicité non connectable, Service Data UUID `08F0`, signal très fort (ici −21 dBm contre le téléphone). La touche **1** a donné l’identifiant `0x148D`.
2. **Installer l’intégration** depuis HACS : `NicoIIT/ha-ble-adv`, redémarrer Home Assistant. HACS n’ajoute pas d’entrée dans la barre latérale. Le chemin est **Paramètres → Appareils et services → Ajouter une intégration**.
3. **Installer ESPHome** dans **Paramètres → Applications** (l’ancien nom est « modules complémentaires »). C’est le module officiel, pas une entrée HACS. Cocher **Afficher dans la barre latérale**.
4. **Flasher la carte** avec [`esphome/relais-ventilateur.yaml`](esphome/relais-ventilateur.yaml). Copier [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example) en `secrets.yaml` et y mettre le Wi-Fi 2,4 GHz. Les secrets sont gravés à la compilation : les changer impose un nouveau flash USB.
5. **Brancher la carte** en USB-C de données. Si aucun port n’apparaît, maintenir **BOOT**, appuyer sur **RST**, relâcher **BOOT**. L’adresse IP utile est celle du journal (`WiFi connected`), pas une adresse devinée.
6. **Ajouter l’intégration** et choisir **Dupliquer la configuration d’une application pour smartphone déjà appairée**. Appuyer sur la touche **1** pendant l’écoute. Laisser le jeu de traducteurs sur **Par défaut**.
7. **Corriger la lumière**, dont le marche-arrêt est inversé sur ce ventilateur. Coller le bloc de [`homeassistant/lumiere-ventilateur.yaml`](homeassistant/lumiere-ventilateur.yaml) dans `configuration.yaml`, dans l’unique clé `template:`.
8. **Pont Maison** : **HomeKit Bridge → Configurer**, cocher `light.lumiere_ventilateur`.

Le journal complet, avec les erreurs rencontrées (mauvais SSID, scan Bluetooth qui étouffait le Wi-Fi, mauvaise adresse IP), est dans [docs/home-assistant.md](docs/home-assistant.md).

### Réglage radio qui a connecté la carte

Un scan Bluetooth continu (fenêtre = intervalle = 20 ms) occupait l’unique antenne du C3. Le Wi-Fi ne trouvait plus le réseau. Le réglage retenu :

```yaml
esp32_ble_tracker:
  scan_parameters:
    interval: 320ms
    window: 30ms
```

Le voyant GPIO8, allumé depuis Home Assistant, confirme que l’API de la carte répond.

### Lumière inversée

Allumer l’entité d’origine éteint la lampe. Le menu **Jeu de traducteurs** de `remote_v21` ne propose pas d’inverser ON/OFF. La correction est une lumière modèle qui échange `turn_on` et `turn_off`. L’entité d’origine reste en place pour la luminosité et la température de couleur.

## Arborescence

```
.
├── README.md                          ce fichier
├── esphome/
│   ├── relais-ventilateur.yaml        firmware du relais ESP32-C3
│   └── secrets.yaml.example           modèle Wi-Fi, à copier en secrets.yaml
├── homeassistant/
│   └── lumiere-ventilateur.yaml       lumière corrigée, à fusionner dans configuration.yaml
└── docs/
    ├── materiel.md                    circuit, photos, schémas, clavier
    ├── protocole.md                   trames BLE et capture nRF Connect
    ├── home-assistant.md              installation du 19 octobre 2026
    ├── photos/
    └── schemas/
```

`secrets.yaml` et les fichiers de compilation ESPHome ne sont pas versionnés (voir `.gitignore`).

## Précautions

- **Ne pas relancer l’appairage mural** tant que la télécommande d’origine fonctionne. La procédure imprimée au dos (courant coupé 60 s, puis **PAIR CODE** dans les 3 s) peut délier cette télécommande.
- Choisir **Dupliquer**, pas **Appairage**, dans l’assistant de l’intégration.
- La carte est un **ESP32-C3**. Un YAML `esp32dev` ne démarre pas dessus.
- Le C3 ne voit que le Wi-Fi **2,4 GHz**.

## Sources

- [ha-ble-adv](https://github.com/NicoIIT/ha-ble-adv) et [esphome-ble_adv_proxy](https://github.com/NicoIIT/esphome-ble_adv_proxy), NicoIIT
- [FanLamp Pro sur l’App Store](https://apps.apple.com/us/app/fanlamp-pro/id1570374552), Shenzhen Jingyuan Micro Control Technology, Android `com.jingyuan.fan_lamp`
- [Clonage d’une télécommande FanLamp](https://garden.c0smere.net/Campsite-Automation/Gazebo-Fans-%E2%80%94-Cloning-the-FanLamp-Remote), Wesley Ray
- Démonte d’une télécommande de la même famille : [Fanbulous TL-FSW071-011](https://www.eevblog.com/forum/reviews/fanbulous-smart-outdoor-ceiling-fan-remote-control-teardown-tl-fsw071-011/)
