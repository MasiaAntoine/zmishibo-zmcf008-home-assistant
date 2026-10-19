# Installation Home Assistant — 19 octobre 2026

Le ventilateur est piloté depuis Home Assistant. La télécommande d’origine reste appairée : on a copié son identifiant, on n’a pas refait l’appairage mural.

Retour au [sommaire](../README.md). Circuit : [materiel.md](materiel.md). Trames : [protocole.md](protocole.md).

## Logiciels

| Rôle | Nom | Où le trouver |
|---|---|---|
| Intégration du ventilateur | `NicoIIT/ha-ble-adv`, titre **BLE ADV Ceiling Fan / Lamps** | HACS, puis redémarrage. Ensuite **Paramètres → Appareils et services → Ajouter une intégration**. HACS n’ajoute pas de menu dans la barre latérale. |
| Compilation de la carte | module **ESPHome**, version utilisée **2026.9.1** | **Paramètres → Applications** (ancien nom : modules complémentaires), pas HACS. Cocher **Afficher dans la barre latérale**. L’éditeur de fichiers était déjà installé au même endroit. |
| Relais radio | `NicoIIT/esphome-ble_adv_proxy` | déclaré dans le YAML ESPHome, pas une intégration HACS |

Home Assistant n’avait aucun adaptateur capable d’émettre ces publicités. Le message au premier essai : « Aucun Adaptateur Bluetooth supportant le BLE Advertising n’a été détecté. » Le Bluetooth du téléphone, celui de nRF Connect, ne compte pas. Un `bluetooth_proxy` classique n’émet pas ces trames. Le relais est la carte.

## Carte

TENSTAR ROBOT Super Mini, **ESP32-C3**, 4 Mo de flash, USB-C natif. Ce n’est pas un ESP32 classique : le YAML `esp32dev` ne démarre pas dessus.

| | |
|---|---|
| Nom ESPHome | `esphome-web-68e5c8` |
| Nom affiché | relais-ventilateur |
| Framework | ESP-IDF, ESP-IDF v5.5.5, puce révision 0.4, flash 4 Mo DIO |
| Compilé | 19 octobre 2026, ESPHome 2026.9.1 |
| MAC Wi-Fi | `DC:06:75:68:E5:C8` |
| MAC Bluetooth | `DC:06:75:68:E5:CA` |
| API | port 6053, chiffrement Noise désactivé |
| Adaptateur vu par l’intégration | `esp/esphome-web-68e5c8` |
| Voyant | GPIO8, inversé. Allumable depuis Home Assistant, c’est le signe que l’API répond |
| Fichier | [`esphome/relais-ventilateur.yaml`](../esphome/relais-ventilateur.yaml) |

Le réseau Wi-Fi s’appelle `masia-antoine.fr`, en **2,4 GHz**. Le secret contenait d’abord `antoine`, un nom qui n’existe pas : la carte répondait `No networks found`, puis `Hidden candidate 'antoine'`, puis `Probe Request Unsuccessful`. Les secrets sont gravés à la compilation. Les modifier oblige à réinstaller par le câble USB.

Modèle à copier : [`esphome/secrets.yaml.example`](../esphome/secrets.yaml.example). Le fichier `secrets.yaml` réel n’est pas dans le dépôt.

Un scan Bluetooth de 20 ms / 20 ms occupait l’antenne en continu et empêchait aussi le Wi-Fi. Des paquets BLE continuaient d’arriver pendant que le Wi-Fi échouait. Le réglage qui a connecté la carte :

```yaml
esp32_ble_tracker:
  scan_parameters:
    interval: 320ms
    window: 30ms
```

Pour le flash : câble USB-C de données. Si aucun port n’apparaît, maintenir **BOOT**, appuyer sur **RST**, relâcher **BOOT**.

Une adoption manuelle vers `192.168.1.20:6053` a échoué (« Impossible de se connecter… section api »). La section `api` était bien dans le YAML : cette adresse n’était pas la carte. L’adresse utile est celle du journal, ligne `WiFi connected`.

La carte reste dans la pièce du ventilateur, alimentée en USB.

## Intégration du ventilateur

Chemin : **Paramètres → Appareils et services → Ajouter une intégration → BLE ADV Ceiling Fan / Lamps**.

1. **Dupliquer la configuration d’une application pour smartphone déjà appairée**. Pas **Appairage**.
2. Appui sur la touche **1** pendant l’écoute. La lumière a clignoté. Configuration retenue : codec **`remote_v21`**, identifiant `0x148D`, index `0`.
3. Entités : lampe principale **Blanc chaud / blanc froid**, seconde lampe **Aucune**, ventilateur **6 niveaux**, sens de rotation activé, oscillation désactivée, mode **Naturel**.
4. Adaptateur : `esp/esphome-web-68e5c8`. Jeu de traducteurs laissé sur **Par défaut**. Délai minimum 750 ms, intervalle 30 ms.
5. Appareil nommé dans la pièce Salon. L’intégration l’affiche `remote_v21 • Salon • 4 entités`.

`remote_v21` correspond à la famille LampSmart v2 télécommande, préfixe `10 00 56`, type d’appareil `0x0100`, sans signature. C’est le même préfixe que la trame de la touche 1.

Le menu **Jeu de traducteurs** de ce codec ne propose pas « Inverser ON/OFF ». Seulement le mode nuit. Le laisser sur **Par défaut**.

## Lumière inversée

Allumer dans Home Assistant éteint la lampe, et l’inverse. Confirmé après la configuration, y compris en laissant le traducteur sur Par défaut. Ce n’est pas un décalage d’état d’un seul appui : chaque commande reste inversée.

L’entité d’origine reste `light.ventilateur_salon_lampe_principale`. Elle sert encore pour la luminosité et la température de couleur. Le bouton marche-arrêt utilisé au quotidien est une lumière modèle. Le fichier prêt à fusionner est [`homeassistant/lumiere-ventilateur.yaml`](../homeassistant/lumiere-ventilateur.yaml) :

```yaml
template:
  - light:
      - name: Lumière ventilateur
        unique_id: lumiere_ventilateur_corrigee
        state: "{{ is_state('light.ventilateur_salon_lampe_principale', 'off') }}"
        turn_on:
          - action: light.turn_off
            target:
              entity_id: light.ventilateur_salon_lampe_principale
        turn_off:
          - action: light.turn_on
            target:
              entity_id: light.ventilateur_salon_lampe_principale
```

Après rechargement de la configuration, l’entité est `light.lumiere_ventilateur`. Elle n’apparaît pas sur la fiche du ventilateur. On la trouve dans **Paramètres → Appareils et services → Entrées**.

S’il existe déjà un bloc `template:` dans `configuration.yaml`, il ne faut pas en coller un second. Les deux lumières se mettent dans le même bloc. Un second bloc du même nom écrase le premier.

La luminosité et la couleur restent sur l’entité d’origine. Le modèle n’envoie pas `set_level` : un `turn_on` sur l’entité d’origine émet la commande `0x10`, qui éteint physiquement la lampe.

## Pont Maison (Apple)

Le pont HomeKit déjà en place n’importe pas les nouvelles entités. Pour voir **Lumière ventilateur** dans l’application Maison : **Paramètres → Appareils et services → HomeKit Bridge → Configurer**, cocher `light.lumiere_ventilateur`, valider, puis rouvrir Maison.
