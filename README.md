# 🚀 Déploiement d'Office 2021 à distance

![Windows](https://img.shields.io/badge/Windows-10%2F11-0078D6?logo=windows&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-Remoting-5391FE?logo=powershell&logoColor=white)
![Office](https://img.shields.io/badge/Office-2021%20LTSC-D83B01?logo=microsoftoffice&logoColor=white)

Guide complet pour installer **Microsoft Office 2021 LTSC** de manière silencieuse sur plusieurs PC d'un réseau, depuis une machine centrale, grâce à l'**Office Deployment Tool (ODT)** et à **PowerShell Remoting**.

---

## 📑 Sommaire

- [Prérequis](#-prérequis)
- [Structure des fichiers](#️-structure-des-fichiers)
- [Étape 1 : Créer le dossier partagé](#-étape-1--créer-le-dossier-partagé)
- [Étape 2 : Télécharger l'ODT](#-étape-2--télécharger-lodt)
- [Étape 3 : Créer le fichier configuration.xml](#-étape-3--créer-le-fichier-configurationxml)
- [Étape 4 : Télécharger les fichiers d'installation](#️-étape-4--télécharger-les-fichiers-dinstallation)
- [Étape 5 : Déployer sur les PC distants](#-étape-5--déployer-sur-les-pc-distants)
- [Vérification](#-vérification)
- [Dépannage](#️-dépannage)
- [Bonnes pratiques](#-bonnes-pratiques)
- [Références](#-références)

---

## 📋 Prérequis

| Élément | Détail |
|---|---|
| **Machine centrale** | Windows 10/11, droits administrateur |
| **PC cibles** | Windows 10/11, accessibles sur le réseau |
| **Espace disque** | ~3 Go pour les fichiers d'installation |
| **Réseau** | Partage SMB fonctionnel entre les machines |
| **Droits** | Administrateur local sur la machine centrale **et** sur les PC cibles |

> ⚠️ **Important :** les PC cibles doivent être allumés et joignables sur le réseau au moment du déploiement. Le nom de la machine centrale doit être résolvable (via DNS ou fichier `hosts`).

---

## 🗂️ Structure des fichiers

```
C:\Office2021\
├── setup.exe              ← Outil officiel Microsoft (ODT)
├── configuration.xml      ← Fichier de configuration
└── Office\                ← Fichiers d'installation téléchargés (~3 Go)
```

---

## 🔧 Étape 1 : Créer le dossier partagé

1. Créez le dossier `C:\Office2021` sur la machine centrale.
2. Clic droit → **Propriétés** → onglet **Partage** → **Partager...**
3. Ajoutez **Tout le monde** avec les droits **Lecture** minimum.
4. Notez le chemin réseau affiché, du type :
   ```
   \\NOM-DE-VOTRE-PC\Office2021
   ```

---

## 📥 Étape 2 : Télécharger l'ODT

1. Téléchargez l'**Office Deployment Tool** depuis le site officiel Microsoft :
   👉 https://www.microsoft.com/download/details.aspx?id=49117
2. Lancez le fichier `.exe` téléchargé.
3. Extrayez son contenu dans `C:\Office2021`.
4. Vous obtenez : `setup.exe` + des fichiers d'exemple.

---

## 📝 Étape 3 : Créer le fichier configuration.xml

Créez un fichier `configuration.xml` dans `C:\Office2021` avec le contenu suivant :

```xml
<Configuration>
  <Add SourcePath="\\NOM-DE-VOTRE-PC\Office2021" OfficeClientEdition="64" Channel="PerpetualVL2021">
    <Product ID="ProPlus2021Volume">
      <Language ID="fr-fr" />
    </Product>
  </Add>
  <Display Level="None" AcceptEULA="TRUE" />
  <Property Name="AUTOACTIVATE" Value="1" />
</Configuration>
```

### 🔁 À personnaliser

| Élément | Description |
|---|---|
| `NOM-DE-VOTRE-PC` | Nom réel de votre machine centrale |
| `ProPlus2021Volume` | Peut être remplacé par `Standard2021Volume` ou `Professional2021Volume` selon votre licence |
| `fr-fr` | Langue d'installation |

> 💡 **Astuce :** pour éviter les problèmes d'encodage, enregistrez le fichier en **UTF-8 sans BOM**.

---

## ⬇️ Étape 4 : Télécharger les fichiers d'installation

Ouvrez l'**Invite de commandes en administrateur** et exécutez :

```cmd
cd C:\Office2021
setup.exe /download configuration.xml
```

> ⏳ Le téléchargement (~3 Go) se fait en arrière-plan. Attendez le retour de l'invite de commandes.

✅ Un dossier `Office\` doit apparaître dans `C:\Office2021`.

---

## 🌐 Étape 5 : Déployer sur les PC distants

Ouvrez **PowerShell en administrateur** sur la machine centrale :

```powershell
$ordinateurs = @("NOM-DU-PC1", "NOM-DU-PC2", "NOM-DU-PC3")

Invoke-Command -ComputerName $ordinateurs -ScriptBlock {
    Start-Process "\\NOM-DE-VOTRE-PC\Office2021\setup.exe" `
        -ArgumentList "/configure \\NOM-DE-VOTRE-PC\Office2021\configuration.xml" `
        -Wait -NoNewWindow
}
```

### 🔁 À personnaliser

- Remplacez `NOM-DU-PC1`, `NOM-DU-PC2`, ... par les vrais noms de vos machines.
- Remplacez `NOM-DE-VOTRE-PC` par le nom de la machine centrale (le même que dans le XML).

Appuyez sur **Entrée** : l'installation démarre en parallèle sur tous les PC listés.

---

## ✅ Vérification

Après 5 à 15 minutes (selon la puissance des PC) :

- Les icônes **Word**, **Excel**, **PowerPoint** apparaissent dans le menu Démarrer des PC cibles.
- Aucune fenêtre ne s'affiche côté utilisateur : l'installation est **100 % silencieuse**.

Pour vérifier depuis la machine centrale :

```powershell
Invoke-Command -ComputerName $ordinateurs -ScriptBlock {
    Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" |
        Where-Object { $_.DisplayName -like "*Office*2021*" } |
        Select-Object DisplayName, DisplayVersion
}
```

---

## 🛠️ Dépannage

| Problème | Solution |
|---|---|
| `Access Denied` sur `Invoke-Command` | Activer WinRM sur les cibles : `Enable-PSRemoting -Force` |
| `setup.exe` introuvable | Vérifier le chemin UNC et les droits de partage |
| Le PC cible n'est pas joignable | Vérifier le nom (DNS/WINS), tester avec `Test-Connection` |
| Installation bloquée | Vérifier `C:\Windows\Temp` et les logs dans `%temp%` |
| Conflit avec une ancienne version d'Office | Désinstaller d'abord avec [SaRA](https://learn.microsoft.com/microsoft-365/troubleshoot/administration/significant-microsoft-365-update-sara) |

---

## 📌 Bonnes pratiques

- 🧪 **Tester sur 1 PC** avant de déployer sur tout un parc.
- 🔐 Ne partager le dossier qu'avec les comptes nécessaires (pas *Tout le monde* en production).
- 📦 Conserver les fichiers `Office\` pour éviter de les re-télécharger à chaque déploiement.
- 🕒 Lancer le déploiement **hors heures de production**.
- 📝 Documenter les noms de machines et la version déployée.

---

## 📚 Références

- [Office Deployment Tool (Microsoft Docs)](https://learn.microsoft.com/deployoffice/overview-office-deployment-tool)
- [Configuration Options for the ODT](https://learn.microsoft.com/deployoffice/configuration-options-for-the-office-2016-deployment-tool)
- [PowerShell Remoting](https://learn.microsoft.com/powershell/scripting/learn/remoting/running-remote-commands)

---

## 🤝 Contribution

Les suggestions et corrections sont les bienvenues : ouvrez une *issue* ou proposez une *pull request*.

---

⭐ **Si ce guide vous a aidé, n'hésitez pas à laisser une étoile sur le repo !**
