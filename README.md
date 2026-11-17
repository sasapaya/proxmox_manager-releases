# ProxMox Manager

Application Android pour gérer un cluster ou un serveur **Proxmox VE** depuis votre téléphone,
**sans service tiers** : elle parle directement à votre Proxmox, rien n'est envoyé ailleurs.

Ce dépôt ne contient que les **versions publiées** (APK signés) et ce guide.

## Télécharger

- Dernière version : **[Releases](../../releases/latest)** → `ProxMoxManager-x.y.z.apk`
- Mises à jour automatiques avec **[Obtainium](https://github.com/ImranR98/Obtainium)** :
  *Ajouter une application* → collez `https://github.com/sasapaya/proxmox_manager-releases` → *Ajouter*.
  Obtainium vous préviendra de chaque nouvelle version et vérifiera qu'elle est signée avec la même clé.

---

## 1. Installer l'application

L'application n'est pas sur le Play Store : elle se distribue sous forme de fichier **APK**.

1. Récupérez le fichier `ProxMoxManager-x.y.z.apk` (et, si possible, le fichier `.sha256` qui l'accompagne).
2. Ouvrez-le sur le téléphone. Android demande d'**autoriser l'installation depuis cette source**
   (navigateur, gestionnaire de fichiers…) : acceptez pour cette application uniquement.
3. **Téléphones Samsung** : si l'installation est bloquée, désactivez temporairement
   *Paramètres → Sécurité et confidentialité → Bloqueur automatique*, installez, puis réactivez-le.

### Vérifier que l'APK est authentique (recommandé)

Toutes les versions sont signées avec la même clé. Empreinte SHA-256 du certificat :

```
085ffb369faa6b7254836f4b4f7a5fbc6cf382046f23ed275fd1df2b2c4a8dc4
```

Depuis un PC avec le SDK Android : `apksigner verify --print-certs ProxMoxManager-x.y.z.apk`
→ la ligne « certificate SHA-256 digest » doit être identique. Android refusera de toute façon
d'installer une mise à jour signée avec une autre clé.

### Mises à jour

Installez simplement le nouvel APK par-dessus l'ancien : vos serveurs et réglages sont conservés.

---

## 2. Préparer Proxmox : un compte dédié

N'utilisez pas `root@pam` depuis un téléphone : créez un compte dédié aux droits limités.
Dans le **Shell** d'un nœud (une seule fois pour tout le cluster) :

```bash
# 1. Le compte
pveum user add mobile@pve
pveum passwd mobile@pve            # choisissez un mot de passe fort

# 2. Un rôle avec les droits utiles à l'application
#    (le nom ne doit PAS commencer par « PVE », réservé à Proxmox)
pveum role add MobileManager -privs "VM.Audit,VM.Console,VM.PowerMgmt,VM.Snapshot,VM.Snapshot.Rollback,VM.Backup,VM.Config.CPU,VM.Config.Memory,VM.Config.Options,VM.Config.Disk,VM.Config.Network,VM.Config.CDROM,VM.Config.HWType,VM.Migrate,VM.Clone,VM.Allocate,Datastore.Audit,Datastore.AllocateSpace,Sys.Audit,SDN.Use"

# 3. Attribution sur tout le cluster
pveum acl modify / -user mobile@pve -role MobileManager
```

Droits **facultatifs**, à n'ajouter que si vous en avez besoin :

| Fonction | Droit | Commande |
|---|---|---|
| Shell des nœuds, haute disponibilité | `Sys.Console` (accès root au nœud !) | `pveum role modify MobileManager -privs "Sys.Console" -append 1` |
| Mises à jour des nœuds | `Sys.Modify` | `pveum role modify MobileManager -privs "Sys.Modify" -append 1` |
| Redémarrer / éteindre un nœud | `Sys.PowerMgmt` | `pveum role modify MobileManager -privs "Sys.PowerMgmt" -append 1` |

Pour une utilisation **en lecture seule** (surveillance uniquement), remplacez le rôle par `PVEAuditor`.

> Le bouton « Mettre à jour » (pveupgrade) peut être réservé par Proxmox à `root@pam`. Sinon :
> *Shell du nœud* → connexion `root` → `apt update && apt full-upgrade`.

---

## 3. Ajouter votre serveur dans l'application

1. **Ajouter un serveur** → saisissez l'adresse : `pve.mondomaine.fr`, ou `192.168.1.10:8006` en local.
2. Choisissez la méthode d'authentification :
   - **Identifiant + mot de passe** (`mobile@pve` + mot de passe) — **recommandé** : nécessaire pour les
     consoles et compatible avec tous les portails d'authentification ;
   - **Token API** (`Datacenter → Permissions → API Tokens`, en décochant *Privilege Separation*) —
     pratique en accès direct, mais ne permet pas d'ouvrir les consoles.
3. **Tester la connexion** (Android 17 et plus : acceptez l'**accès au réseau local** demandé au premier
   test, indispensable pour joindre un Proxmox en `192.168.x.x` ou `10.x`) :
   - *Certificat non reconnu* : normal pour le certificat d'origine de Proxmox. Comparez l'empreinte avec
     celle affichée dans *Nœud → Système → Certificats*, puis **Faire confiance**. Si elle change un jour,
     l'application vous alertera.
   - *Portail d'authentification détecté* : voir la section 4.
4. **Enregistrer**.

---

## 4. Proxmox derrière un reverse proxy et un portail (Authelia, Authentik…)

L'application sait se connecter au portail : bouton **« Se connecter au portail »**, identifiez-vous
comme dans un navigateur. La session est conservée chiffrée sur le téléphone.

- Utilisez la méthode **Identifiant + mot de passe** : certains portails (dont Authelia par défaut)
  refusent les requêtes portant un token API. L'application le détecte et vous le signale.
  Pour garder les tokens, il faut configurer dans Authelia un endpoint d'autorisation « cookie uniquement » pour Proxmox.
- Les sessions de portail expirent (souvent après quelques minutes d'inactivité) : l'application
  propose alors de se reconnecter. Cochez **« Se souvenir de moi »** sur le portail pour l'éviter.
- Les consoles (noVNC, terminal) passent par le reverse proxy : il doit relayer les **WebSockets**
  (c'est le cas si la console fonctionne déjà dans votre navigateur).

---

## 5. Sécurité

- **Verrouillage** par empreinte / visage / code du téléphone à l'ouverture et après une absence
  (réglable dans *Paramètres → Sécurité*). Le téléphone doit avoir un verrouillage d'écran.
- Les opérations **destructrices** (forcer l'arrêt, restaurer, supprimer, convertir en modèle,
  redémarrer un nœud…) demandent une confirmation **et** votre empreinte.
- Configuration, mots de passe et sessions sont **chiffrés** sur le téléphone (Android Keystore)
  et exclus des sauvegardes Android. Pas de capture d'écran possible dans l'application.
- **Se déconnecter** (menu ⋮ du tableau de bord) efface la session du portail et la connexion Proxmox.
- *Paramètres → Journal des actions* : historique local de ce que vous avez fait depuis l'application.

---

## 6. Fonctions principales

| Où | Quoi |
|---|---|
| Tableau de bord | Cluster, quorum, nœuds, stockages, VM et conteneurs (recherche, filtres) ; bouton **Créer** |
| Fiche VM / conteneur | **Résumé** (actions, console, IP), **Performances** (graphiques), **Snapshots**, **Sauvegardes**, **Gérer** (ressources, disques, clonage, migration, HA, modèle) |
| Écran nœud | Version, charge, graphiques, shell, mises à jour, redémarrage / arrêt |
| Icône ☰ du tableau de bord | Tâches Proxmox en cours et récentes, avec leur journal |
| Paramètres | Thème, verrouillage, notifications, journal des actions, export / import de la configuration |
| Écran d'accueil du téléphone | Widget de synthèse : état, nœuds, machines, CPU, RAM, stockage par serveur (sans nom de machine) |
| Écran d'accueil du téléphone | Widget « Machines Proxmox » : chaque VM et conteneur avec son état, CPU et RAM (affiche les noms) |
| Appui long sur un widget → Reconfigurer | Choisir les serveurs affichés dans ce widget |

### Hors ligne

Le dernier état connu de chaque serveur est conservé (chiffré) : sans réseau, le tableau de bord
l'affiche avec la mention « Hors ligne ».

### Changer de téléphone

*Paramètres → Export / import de la configuration* : exportez vos serveurs (et, au choix, leurs mots de
passe) dans un fichier **chiffré par une phrase secrète**, puis importez-le sur le nouveau téléphone.
Sans la phrase secrète, le fichier est inexploitable. Les sessions de portail ne sont jamais exportées.

### Consoles

- **VM** : console graphique noVNC. Boutons ⌨ (clavier), *Ctrl+Alt+Suppr*, *100 % / Ajuster*.
- **Clavier AZERTY** : avec le clavier du téléphone, les lettres arrivent en QWERTY tant que Proxmox ne
  connaît pas votre disposition. Une fois pour tout le cluster : `pvesh set /cluster/options --keyboard fr`
  (ou *Datacenter → Options → Keyboard Layout*), puis arrêtez / démarrez les VM depuis Proxmox.

### Notifications

*Paramètres → Notifications* : l'application vérifie vos serveurs en arrière-plan (toutes les 15 min à 3 h)
et vous prévient en cas de nœud hors ligne, de stockage presque plein ou de tâche en échec
(sauvegarde ratée…). Chaque problème n'est signalé qu'une fois. Derrière un portail, une notification
vous invite à vous reconnecter quand sa session a expiré.

---

## 7. En cas de problème

| Message | Que faire |
|---|---|
| Droits insuffisants : Permission check failed (…, **VM.Xxx**) | Ajoutez le droit cité au rôle `MobileManager` (`pveum role modify … -append 1`) |
| Identifiants refusés | Vérifiez `utilisateur@royaume` (ex. `mobile@pve`) et le mot de passe dans la fiche serveur |
| Le portail bloque le token Proxmox | Passez en méthode **Identifiant + mot de passe** |
| Le certificat a changé | Normal après un renouvellement ; sinon, méfiance. Approuvez seulement si vous savez pourquoi |
| Console noire / clavier inopérant | Bouton ⌨ ; pour une VM, *100 %* et disposition `keyboard fr` (section 6) |
| Serveur injoignable | Réseau du téléphone, VPN, adresse et port (`:8006` en accès direct) |
| Délai dépassé en réseau local alors que le navigateur fonctionne | Android 17 : bouton **Autoriser l'accès au réseau local** (ou *Paramètres Android → Applications → ProxMox Manager → Autorisations*) |
