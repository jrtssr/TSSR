

## Principe
- **WDS** (Windows Deployment Services) : serveur qui installe Windows sur des postes **via le réseau** (boot **PXE**), sans ISO ni clé USB.
- **Mode autonome** : WDS fonctionne **sans Active Directory**.
- **DHCP** : obligatoire. Il donne l'IP au client et l'aiguille vers WDS.
- **boot.wim** : image de démarrage (lance l'installeur Windows).
- **install.wim** : image du système à installer.

## Schéma du labo
```
[VM Client Win10 vide] --PXE--> [VM Windows Server : DHCP + DNS + WDS]
            réseau VMware NAT (VMnet8) - 192.168.202.0/24
```

| Élément | Valeur |
|---|---|
| Sous-réseau VMnet8 | 192.168.202.0 / 255.255.255.0 |
| IP du serveur | 192.168.202.10 |
| Passerelle (NAT VMware) | 192.168.202.2 |
| DNS | 192.168.202.10 et 8.8.8.8 |
| Étendue DHCP | 192.168.202.100 à 192.168.202.200 |

---

## Étape 1 : Réseau VMware
1. VMware : **Edit > Virtual Network Editor > Change Settings**.
2. Sélectionner **VMnet8 (NAT)**.
3. **Décocher** « Use local DHCP service... ».
4. Noter le sous-réseau, puis **OK**.
5. Serveur **et** client sur **VMnet8 (NAT)**.

**Ne pas oublier :** un seul DHCP sur le réseau, donc celui de VMware doit être décoché.

## Étape 2 : IP fixe du serveur
1. `Windows + R` > `ncpa.cpl`.
2. Ethernet0 > Propriétés > **IPv4**.
3. IP `192.168.202.10`, masque `255.255.255.0`, passerelle `192.168.202.2`.
4. DNS préféré `192.168.202.10`, auxiliaire `8.8.8.8`.
5. Tester : `ping 192.168.202.2` puis `ping 8.8.8.8`.

## Étape 3 : Installer les rôles
1. **Gestionnaire de serveur > Gérer > Ajouter des rôles et fonctionnalités**.
2. Cocher : **Services de déploiement Windows**, **Serveur DHCP** (+ DNS si voulu).
3. Accepter les fonctionnalités requises.
4. Laisser cochés les **deux services de rôle WDS**, puis **Installer**.
5. **Drapeau jaune > Terminer la configuration DHCP > Valider**.

Alternative PowerShell :
```
Install-WindowsFeature wds-deployment -IncludeManagementTools
```

## Étape 4 : Créer l'étendue DHCP
1. **Outils > DHCP**.
2. Clic droit sur **IPv4 > Nouvelle étendue**.
3. Nom au choix, plage `192.168.202.100` à `192.168.202.200`, longueur `24`.
4. Pas d'exclusions, bail par défaut.
5. Options : **routeur `192.168.202.2`**, **DNS `192.168.202.10`**.
6. **Activer l'étendue** (flèche verte).
7. Vérifier dans **Options d'étendue** : 003 Routeur = `192.168.202.2`.

**Ne pas oublier :** vérifier la valeur du routeur (faute de frappe possible).

## Étape 5 : Prévoir un second disque ou une partition
1. `diskmgmt.msc` > clic droit sur C: > **Réduire le volume**.
2. Créer un nouveau volume NTFS (ex. `F:`).
3. Sinon, `C:\RemoteInstall` fonctionne en labo (accepter l'avertissement).

## Étape 6 : Configurer WDS
1. Console WDS : **Outils > Services de déploiement Windows**.
2. Clic droit sur le serveur > **Configurer le serveur**.
3. Cocher **Serveur autonome**.
4. Dossier d'installation à distance : `F:\RemoteInstall` (ou `C:\RemoteInstall`).
5. Écouteur DHCP : **cocher les deux cases**.
6. Réponse PXE : **Répondre à tous les ordinateurs clients**.
7. Terminer en **décochant** « Ajouter des images maintenant ».
8. Clic droit sur le serveur > **Toutes les tâches > Démarrer**.
9. Icône **verte** = service actif.

**Ne pas oublier :**
- Les deux cases de l'écouteur DHCP doivent être cochées.
- Le service WDS se démarre à la main après la configuration.
- Si le chemin est refusé : cocher **Partage de fichiers et d'imprimantes Réseaux Microsoft** dans les propriétés de la carte réseau.

## Étape 7 : Ajouter les images
Monter l'**ISO Windows 10** dans la VM serveur (lecteur `D:`), dossier `sources`.

**Image de démarrage**
1. Clic droit sur **Images de démarrage > Ajouter une image de démarrage**.
2. Parcourir `D:\sources\boot.wim`, donner un nom (ex. `Win10-Boot`), terminer.

**Image d'installation**
1. Clic droit sur **Images d'installation > Ajouter une image d'installation**.
2. Créer un groupe d'images (ex. `Windows10`).
3. Parcourir `D:\sources\install.wim`.
4. Garder seulement l'édition voulue (ex. **Pro**), donner un nom reconnaissable.
5. Attendre l'import, terminer.

**Ne pas oublier :**
- Si l'ISO n'a qu'un `install.esd`, le convertir :
```
dism /Get-WimInfo /WimFile:D:\sources\install.esd
dism /Export-Image /SourceImageFile:D:\sources\install.esd /SourceIndex:<N> /DestinationImageFile:C:\install.wim /Compress:max /CheckIntegrity
```
- Si l'image n'apparaît pas au déploiement, le `.wim` est probablement corrompu : le réimporter depuis l'ISO officielle.

## Étape 8 : Créer la VM cliente
1. VMware : **New Virtual Machine > Custom**.
2. **I will install the operating system later**.
3. Système : **Windows 10 x64**, firmware **BIOS**.
4. **RAM : 4 Go minimum**, disque 60 Go (SATA ou LSI SAS).
5. Réseau : **NAT (même VMnet8 que le serveur)**.
6. Carte réseau **E1000E** : dans le `.vmx`, ajouter `ethernet0.virtualDev = "e1000e"`.

**Ne pas oublier :** aucune ISO dans la VM cliente.

## Étape 9 : Tester le déploiement (PXE)
1. Démarrer la VM cliente vide.
2. Elle reçoit une IP entre `.100` et `.200`.
3. Si besoin, **F12** pour choisir le démarrage réseau (LAN/PXE).
4. À « Press F12 for network service boot », **F12 de nouveau**.
5. Le client télécharge `boot.wim`, l'installeur démarre.
6. Identifiants : `NOM_SERVEUR\Administrateur` + mot de passe.
7. Choisir l'image **Windows 10 Pro**, le disque, puis lancer l'installation.

**Ne pas oublier :** il faut être rapide sur F12 ; sinon éteindre et rallumer la VM.