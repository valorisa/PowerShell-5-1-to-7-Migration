# Tableau de compatibilité et matrice de décision pour la migration de Windows PowerShell 5.1 vers PowerShell 7

Documentation de référence pour évaluer un parc Windows avant de faire de PowerShell 7 (`pwsh.exe`) l'environnement par défaut : différences techniques, compatibilité des modules, changements de syntaxe, règles d'exécution côte à côte et checklist d'audit.

> **Statut du contenu.** Synthèse dérivée d'un article de presse et de documentation Microsoft Learn, compilée à l'origine via un outil de recherche assisté par IA, puis partiellement recoupée avec des pages Microsoft le 24 novembre 2026. Ce qui est confirmé, corrigé ou reste à vérifier est détaillé dans [Limites](#limites-et-points-à-vérifier). À confronter à la documentation officielle avant toute décision de production.

## Récupérer le dépôt

```bash
git clone https://github.com/valorisa/PowerShell-5-1-to-7-Migration.git
cd PowerShell-5-1-to-7-Migration
```

## Sommaire

- [Récupérer le dépôt](#récupérer-le-dépôt)
- [Contexte](#contexte)
- [Différences essentielles](#différences-essentielles)
- [Points de vigilance](#points-de-vigilance)
- [Matrice de migration](#matrice-de-migration)
- [Compatibilité des modules](#compatibilité-des-modules)
- [Couche Windows Compatibility](#couche-windows-compatibility)
- [Changements de syntaxe](#changements-de-syntaxe)
- [Règles côte à côte (side-by-side)](#règles-côte-à-côte-side-by-side)
- [Checklist d'audit](#checklist-daudit)
- [Gabarit de décision](#gabarit-de-décision)
- [Stratégie de migration recommandée](#stratégie-de-migration-recommandée)
- [Limites et points à vérifier](#limites-et-points-à-vérifier)
- [Sources](#sources)
- [Licence](#licence)

## Contexte

Microsoft a annoncé que PowerShell 7 sera inclus par défaut dans la prochaine version de Windows, et que Windows PowerShell 5.1 est déprécié. Les deux versions continuent de coexister : PowerShell 7 ne remplace pas 5.1.

- Windows 10 et Windows 11 embarquent aujourd'hui PowerShell 5.1. D'après l'annonce de Microsoft relayée par la presse, Windows Server 2025 et Windows 11 26H2 sont les dernières versions à l'activer par défaut.
- Dans la prochaine version de Windows, PowerShell 7 sera inclus et 5.1 deviendra une fonctionnalité à la demande (activée par défaut sur les nouvelles installations). Il disparaîtra ensuite d'une version ultérieure, dont le calendrier sera communiqué à l'avance.
- Microsoft n'a pas nommé cette prochaine version. 27H2 (second semestre 2027) est une estimation de la presse, pas une date confirmée.
- PowerShell 7.6 LTS est indiqué comme supporté jusqu'au 14 novembre 2028 : information issue du PDF source, non recoupée. PowerShell 7.7 est sorti le 10 novembre 2026.

Un script qui s'exécute sans erreur sous PowerShell 7 peut produire un résultat différent de celui obtenu sous 5.1. La migration doit donc être validée par des tests, pas seulement par une exécution réussie.

## Différences essentielles

| Aspect | Windows PowerShell 5.1 | PowerShell 7 |
|---|---|---|
| Runtime | .NET Framework (v4.5 selon Microsoft Learn) | .NET moderne (anciennement .NET Core) |
| Plateformes | Windows uniquement | Windows, macOS, Linux |
| Installation | Composant intégré à Windows | Installation séparée, coexiste avec 5.1 |
| Modules Windows | Accès natif à l'écosystème Windows | Beaucoup fonctionnent nativement, d'autres via la couche de compatibilité |
| Performances et syntaxe | Base ancienne | Pipeline plus rapide, opérateur ternaire, chaînage de pipeline `&&` / `\|\|`, améliorations des erreurs et du JSON |
| Support | Lié au cycle de vie de Windows ; déprécié | Cycle de vie aligné sur .NET ; 7.6 LTS supporté jusqu'au 14 novembre 2028 |

## Points de vigilance

**Fonctionnalités supprimées ou modifiées**

- PowerShell Workflow
- Première version de WMI (WMI v1)
- Certains cmdlets DSC
- `New-WebServiceProxy`
- DCOM pour `Restart-Computer` / `Stop-Computer`

**Cmdlets web**

- `Invoke-WebRequest` et `Invoke-RestMethod` reposent désormais sur `HttpClient`.
- `ParsedHtml` et `Forms` ne sont plus pris en charge de la même manière.
- Les schémas `file://` et `ftp://` ne sont plus supportés de la même manière.

**Encodage**

- `$OutputEncoding` passe à UTF-8 sans BOM.
- `-Encoding Byte` devient `-AsByteStream`.
- Ces changements peuvent casser des traitements de fichiers ou des échanges avec des outils externes.

### Distribution MSIX (points à tester)

PowerShell 7 est livré dans Windows sous forme de paquet MSIX. D'après la presse (Next, Techzine) :

- l'installation se fait dans un dossier protégé en lecture seule et s'enregistre par utilisateur, là où le MSI s'installait pour toute la machine ;
- trois points restent ouverts côté Microsoft : l'exécution sous le compte SYSTEM (tâches planifiées, services, gestion d'appareils, remoting), la prise en charge des cmdlets DISM comme `Add-WindowsCapability`, et les paramètres à l'échelle de la machine ;
- les stratégies de journalisation (modules, blocs de script, transcription) sont à redéployer pour PS7, et une supervision basée sur les journaux de 5.1 ne verra pas l'activité de PS7 sans adaptation.

Ces éléments viennent de sources secondaires : à confirmer dans la documentation Microsoft.

## Matrice de migration

| Domaine | Windows PowerShell 5.1 | PowerShell 7 | Implication de migration |
|---|---|---|---|
| Exécutable | `powershell.exe` | `pwsh.exe` | Mettre à jour raccourcis, planificateurs, agents, wrappers et shebangs. `-File` est désormais le premier paramètre positionnel de `pwsh`. |
| Runtime | .NET Framework | .NET moderne, selon la version de PowerShell 7 | Tester les appels .NET directs, surcharges de méthodes, types et assemblies (`[type]::Method()`, `New-Object`). |
| Plateformes | Windows uniquement | Windows, macOS, Linux | Permet de standardiser les scripts multi-OS ; isoler les appels spécifiques à Windows. |
| Installation | Intégré à Windows | Installation séparée, côte à côte | Ne pas désinstaller 5.1 avant d'avoir validé toutes les dépendances. |
| Profils | `$HOME\Documents\WindowsPowerShell` | `$HOME\Documents\PowerShell` | Profils non partagés : recréer, auditer et versionner les profils PS7. |
| Modules utilisateur | `$HOME\Documents\WindowsPowerShell\Modules` | `$HOME\Documents\PowerShell\Modules` | Les modules installés pour 5.1 ne sont pas automatiquement disponibles dans 7. |
| Modules machine | `%ProgramFiles%\WindowsPowerShell\Modules` | `%ProgramFiles%\PowerShell\Modules` | Prévoir un déploiement distinct des modules AllUsers pour PS7. |
| Modules système | `%SystemRoot%\System32\WindowsPowerShell\1.0\Modules` | Inclut `%windir%\System32` au démarrage, mais n'expose que les modules marqués compatibles Core | Un module système peut être invisible ou refusé dans PS7 selon son manifeste. |
| Encodage | Souvent Windows-1252 / ASCII selon le contexte | `$OutputEncoding` par défaut en UTF-8 sans BOM | Vérifier sorties, fichiers CSV, logs, accents et intégrations avec les outils externes. |
| Erreurs natives | Avec `2>&1`, `$?` passe à faux dès que l'exécutable écrit sur stderr, même avec le code de sortie 0 (constaté en test, voir [Limites](#limites-et-points-à-vérifier)) | `$?` suit le code de sortie : vrai si 0, faux sinon | Un contrôle fondé sur `$?` qui échouait sur stderr en 5.1 ne le fera plus en 7 : contrôler `$LASTEXITCODE`. Avant PS7, `$?` était aussi remis à vrai dans `(...)`, `$(...)` et `@(...)`. |
| CSV | `Export-Csv` inclut les informations de type | `-NoTypeInformation` est le comportement par défaut | Ajouter `-IncludeTypeInformation` si un consommateur dépend de l'en-tête de type. |
| Web | `Invoke-WebRequest` / `Invoke-RestMethod` sur l'ancienne pile .NET | Fondés sur `HttpClient` ; parsing HTML simplifié, en-têtes plus stricts, FTP et `file://` non supportés | Auditer les scripts d'API, de téléchargement, de formulaires et d'authentification. |
| Remoting DCOM | Certains cmdlets acceptaient `-Protocol DCOM` | DCOM n'est plus supporté pour le remoting | Migrer vers WinRM/PSRP ou `Invoke-Command`. |
| Snap-ins | `Add-PSSnapin`, etc. | Retirés | Remplacer les snap-ins par des modules. |
| WMI v1 | `Get-WmiObject`, `Invoke-WmiMethod`, etc. | Retirés | Migrer vers les cmdlets CIM (`Get-CimInstance`, `Invoke-CimMethod`, etc.). |

## Compatibilité des modules

Trois statuts sont possibles pour un module dans PowerShell 7 : **natif**, **via la couche de compatibilité** (`-UseWindowsPowerShell`), ou **retiré**.

| Module | Statut dans PowerShell 7 | Action requise |
|---|---|---|
| `ActiveDirectory` | Nativement compatible avec les prérequis appropriés : Windows Server 1809+ avec `RSAT-AD-PowerShell`, ou Windows 10 1809+ avec `Rsat.ActiveDirectory.DSLDS.Tools`. | Installer/valider les RSAT, puis tester `Get-ADUser`, `Get-ADComputer`, `Get-ADGroup` et les opérations d'écriture sous `pwsh`. |
| `Appx` | Nativement compatible, mais la compatibilité avec PowerShell 7.1 exige la couche de compatibilité selon la documentation Microsoft. | Valider la version PS7 cible et les cmdlets `Get-AppxPackage`, `Add-AppxPackage`, `Remove-AppxPackage` dans l'environnement réel. |
| `ISE` | Retiré de PowerShell 7 ; ISE ne supporte que Windows PowerShell. | Remplacer ISE par VS Code + extension PowerShell ; ne pas prévoir `Import-Module ISE` dans PS7. |
| `PSScheduledJob` | Non supporté par la couche de compatibilité ; module intégré à Windows PowerShell 5.1 et explicitement sur la deny list. | Réécrire avec `ScheduledTasks`, le Planificateur de tâches Windows, des jobs PS7 ou un outil d'orchestration ; ne pas compter sur `-UseWindowsPowerShell`. |
| `PSWorkflow` | Retiré de PowerShell 7 ; PowerShell Workflow repose sur Windows Workflow Foundation, absent de .NET moderne. Marqué « untested with compatibility layer » dans le tableau Microsoft. | Réécrire : `ForEach-Object -Parallel` pour le parallélisme simple, Planificateur de tâches pour la reprise après redémarrage, persistance d'état explicite si nécessaire. |
| `PSWorkflowUtility` | Retiré de PowerShell 7. | Migrer avec `PSWorkflow`. |
| `Microsoft.PowerShell.LocalAccounts` | N'est plus livré avec PowerShell 7, mais Microsoft le classe « Natively Compatible » sur Windows 10 / Server 1809+ (module fourni par Windows). | Tester `Get-LocalUser` et `Get-LocalGroup` sous `pwsh` sur la version de Windows cible ; ne prévoir une alternative que si le module ne se charge pas. |
| `Microsoft.PowerShell.ODataUtils` | Retiré de PowerShell 7. | Remplacer par une intégration OData/REST moderne ou conserver 5.1 temporairement. |
| `Microsoft.PowerShell.Operation.Validation` | Retiré de PowerShell 7. | Remplacer par Pester ou un framework de tests dédié. |
| Modules WMI v1 | Retirés : `Get-WmiObject`, `Invoke-WmiMethod`, `Set-WmiInstance`, `Remove-WmiObject`, `Register-WmiEvent`. | Migrer vers CIM v2. |
| `*-EventLog` | Plusieurs cmdlets retirés. | Migrer vers `Get-WinEvent` et `New-WinEvent`. |
| DSC (`PSDesiredStateConfiguration`) | Retiré du produit à partir de PowerShell 7.2 et publié dans la PowerShell Gallery. | Gérer DSC comme dépendance de package, pas comme module intégré. |

**Décision recommandée**

- `ActiveDirectory` peut généralement être migré nativement.
- `Appx` doit être validé selon la version de PowerShell 7 ciblée.
- `ISE`, `PSScheduledJob` et `PSWorkflow` sont à traiter comme des dépendances 5.1 ou à refondre.

## Couche Windows Compatibility

`Import-Module -UseWindowsPowerShell` crée un module proxy dans PowerShell 7 qui délègue les cmdlets à un processus local Windows PowerShell 5.1 (implicit remoting). C'est une solution de transition, pas une compatibilité native.

```powershell
Import-Module -Name ScheduledTasks -UseWindowsPowerShell
```

La couche se déclenche aussi de façon implicite : l'import par nom ou par chemin, ou le chargement automatique d'un module Windows PowerShell non marqué compatible Core, passe par le même mécanisme. Ce comportement se désactive avec le paramètre `DisableImplicitWinCompat` du fichier `powershell.config.json`.

| Règle | Conséquence |
|---|---|
| Fonctionne uniquement en local sur Windows | Ne pas l'utiliser pour une automatisation Linux/macOS ni pour un scénario distant. |
| Exige Windows PowerShell 5.1 | 5.1 doit rester installé et opérationnel. |
| Sérialise paramètres et résultats | Les objets retournés ne sont pas des objets vivants : méthodes, propriétés dynamiques et références d'objet peuvent être perdues ou modifiées. |
| Partage un runspace entre les modules importés dans la session de compatibilité | États, variables et effets de bord peuvent interagir entre modules. |
| Respecte une deny list | Valeur par défaut de `WindowsPowerShellCompatibilityModuleDenyList` : `PSScheduledJob`, `BestPractices` et `UpdateServices`. |
| N'est pas un substitut à la compatibilité native | À réserver aux dépendances à court terme, avec un plan de remplacement. |

La latence et les différences de comportement induites par le processus 5.1 en arrière-plan doivent être mesurées.

## Changements de syntaxe

Les constructions ci-dessous ne cassent pas le code 5.1 : elles sont **indisponibles en 5.1** et **disponibles en PowerShell 7**. Le risque est inverse : un script PS7 qui les utilise échouera s'il est exécuté par `powershell.exe`.

| Constructeur | PowerShell 5.1 | PowerShell 7 | Exemple PS7 |
|---|---|---|---|
| Ternaire `? :` | Non disponible | Disponible | `$result = $ok ? 'OK' : 'Failed'` |
| Null-coalescing `??` | Non disponible | Disponible | `$value = $config ?? 'default'` |
| Null-coalescing assignment `??=` | Non disponible | Disponible | `$config ??= 'default'` |
| Null-conditional `?.` | Non disponible | Disponible depuis 7.1 | `${obj}?.Property` |
| Null-conditional element `?[]` | Non disponible | Disponible depuis 7.1 | `${items}?[0]` |
| Chaînage de pipeline `&&` | Non disponible | Disponible | `Test-Path $path && Copy-Item a b` |
| Chaînage de pipeline `\|\|` | Non disponible | Disponible | `Test-Path $path \|\| Write-Warning 'Missing'` |
| Job en fin de pipeline `&` | Non disponible sous cette forme | Disponible | `Get-Process &` retourne un job PowerShell |
| `ForEach-Object -Parallel` | Non disponible | Disponible depuis 7.0 | `$items \| ForEach-Object -Parallel { ... } -ThrottleLimit 5` |

**Règles de portabilité**

- Script devant fonctionner en 5.1 **et** en 7 : éviter `?:`, `??`, `??=`, `&&`, `||`, le `&` de job et `-Parallel`.
- Script PS7 uniquement : ajouter `#requires -Version 7` en tête de fichier pour échouer rapidement sous 5.1.
- Avec `?.` et `?[]` : utiliser la forme explicite `${variable}`, car `?` peut faire partie d'un nom de variable.
- `&&` et `||` enchaînent des pipelines PowerShell ; ils ne sont pas équivalents à `$LASTEXITCODE`, qui concerne le code de sortie du dernier programme natif.

## Règles côte à côte (side-by-side)

### PSModulePath

PowerShell 7 construit son chemin de modules dans l'ordre suivant : modules CurrentUser PS7, modules système PS7, puis `$PSHOME\Modules`. Il ajoute aussi `%windir%\System32`, mais n'expose que les modules dont le manifeste est compatible avec l'édition Core.

| Scénario | Comportement attendu |
|---|---|
| Lancer `pwsh` depuis 5.1 | PS7 préfixe ses chemins ; les chemins hérités de 5.1 restent présents, sans problème fonctionnel signalé. |
| Lancer `powershell.exe` directement depuis PS7 | PS7 copie sa valeur dans `WinPSModulePath`, retire ses chemins PS7, puis démarre 5.1 avec les chemins Windows PowerShell corrects. |
| Lancer `powershell.exe` via un processus intermédiaire lancé par PS7 | Le processus intermédiaire peut transmettre le `PSModulePath` PS7 à 5.1 ; un module commun peut alors se résoudre vers une version PS7 incompatible et provoquer une `CommandNotFoundException`. |
| Plusieurs versions d'un même module | PowerShell charge par défaut la version la plus élevée ; utiliser `Import-Module -FullyQualifiedName` pour épingler une version. |

**Correction pour le processus intermédiaire** : supprimer `PSModulePath` de l'environnement du processus enfant. Utiliser `cmd.exe` avec son extension : si un fichier `cmd` sans extension précède `cmd.exe` dans le `PATH` (cas d'une installation MSYS2, constaté lors des tests), `cmd` seul ouvre une fenêtre de sélection d'application au lieu de lancer l'interpréteur.

```cmd
cmd.exe /c "set PSModulePath=&& powershell.exe -File script.ps1"
```

### Profils

| Profil | Windows PowerShell 5.1 | PowerShell 7 |
|---|---|---|
| Utilisateur courant | `$HOME\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1` | `$HOME\Documents\PowerShell\Microsoft.PowerShell_profile.ps1` |
| Tous les utilisateurs | `$PSHOME\profile.ps1` | `$PSHOME\profile.ps1`, dans l'installation PS7 |
| Hôte spécifique | `Microsoft.PowerShellISE_profile.ps1` pour ISE | Profil spécifique à l'hôte si applicable |

Ne pas copier aveuglément le profil 5.1 dans PS7 : il peut charger des modules incompatibles, définir des alias qui masquent des cmdlets PS7, ou dépendre de composants Windows absents.

## Checklist d'audit

### Inventaire

- [ ] Recenser tous les fichiers `.ps1`, `.psm1`, `.psd1`, `.ps1xml`, modules, fonctions et profils utilisés.
- [ ] Identifier les hôtes d'exécution : console interactive, ISE, Planificateur de tâches, services Windows, SCCM/Intune, CI/CD, agents de supervision, RMM.
- [ ] Identifier les comptes d'exécution, leur profil chargé et leurs variables d'environnement.
- [ ] Recenser les dépendances : modules Microsoft, éditeurs tiers, DLL .NET Framework, COM, WMI, snap-ins, exécutables externes.
- [ ] Cartographier les scripts appelés depuis `powershell.exe`, `powershell -Command`, `powershell -File`, `powershell_ise.exe` ou des fichiers `.bat` / `.cmd`.

### Modules

- [ ] Exécuter un inventaire des modules dans 5.1 et dans 7 : `Get-Module -ListAvailable`.
- [ ] Comparer les versions disponibles dans les deux environnements.
- [ ] Classer chaque module : natif PS7, compatible via `-UseWindowsPowerShell`, à remplacer, ou 5.1 uniquement.
- [ ] Vérifier `CompatiblePSEditions` dans les manifestes `.psd1` des modules internes.
- [ ] Tester explicitement `Import-Module ActiveDirectory` et les cmdlets AD réellement utilisés.
- [ ] Rechercher dans le code : `PSScheduledJob`, `PSWorkflow`, `PSWorkflowUtility`, `ISE`, `Get-WmiObject`, `Add-PSSnapin`, `*-EventLog`.
- [ ] Remplacer les jobs planifiés `PSScheduledJob` par `ScheduledTasks`, le Planificateur Windows ou une solution moderne.
- [ ] Remplacer les workflows par `ForEach-Object -Parallel`, des jobs, des files d'attente ou un orchestrateur selon le besoin de reprise et de persistance.

### Syntaxe et exécution

- [ ] Scanner les scripts pour les opérateurs PS7 uniquement : `?:`, `??`, `??=`, `&&`, `||`, `&` en fin de pipeline, `-Parallel`.
- [ ] Scanner les appels .NET directs et valider les surcharges, notamment les méthodes de chaînes et de collections.
- [ ] Corriger les appels WMI v1 vers CIM.
- [ ] Remplacer les snap-ins par des modules.
- [ ] Revoir les scripts qui dépendent de `$?` après une commande native écrivant sur stderr.
- [ ] Revoir `Export-Csv` si les fichiers produits doivent conserver l'en-tête de type.
- [ ] Tester les encodages de fichiers : UTF-8, BOM, accents, sorties d'outils externes.
- [ ] Valider les webhooks, API REST, téléchargements, en-têtes personnalisés et authentifications HTTP.
- [ ] Vérifier chemins, lecteurs mappés, redirections Documents/OneDrive et emplacements de profils.

### Côte à côte

- [ ] Confirmer que `pwsh.exe` et `powershell.exe` sont tous deux disponibles et versionnés.
- [ ] Vérifier `$PSVersionTable`, `$PSHOME`, `$PROFILE` et `$Env:PSModulePath` dans les deux shells.
- [ ] Vérifier qu'aucun module PS7 n'est résolu par 5.1 lors d'un lancement via processus intermédiaire.
- [ ] Ajouter des contrôles de version : `#requires -Version 7` pour PS7 uniquement, ou une garde de version explicite pour les scripts hybrides.
- [ ] Mettre à jour les lanceurs pour appeler explicitement `pwsh.exe` lorsque PS7 est requis.
- [ ] Ne pas modifier globalement `PSModulePath` sans tester l'effet sur les deux éditions et sur les processus enfants.
- [ ] Documenter les modules conservés en 5.1 : propriétaire, raison, date de retrait prévue.

### Distribution, comptes et journalisation

- [ ] Vérifier le mode d'installation de PS7 (MSIX ou MSI, par utilisateur ou par machine) sur le parc.
- [ ] Tester les scripts sous le compte SYSTEM : tâches planifiées, services, gestion d'appareils, remoting.
- [ ] Tester les cmdlets DISM (par exemple `Add-WindowsCapability`) sous `pwsh`.
- [ ] Redéployer les stratégies de journalisation PS7 (modules, blocs de script, transcription) et adapter la supervision, qui ne voit pas l'activité de PS7 si elle lit uniquement les journaux de 5.1.
- [ ] Décider si le chargement implicite de la couche de compatibilité doit être désactivé (`DisableImplicitWinCompat`).
- [ ] Planifier le cycle de vie : 5.1 devient une fonctionnalité à la demande puis disparaît ; les versions LTS de PS7 ont un support limité dans le temps.

### Validation

- [ ] Exécuter les tests dans un environnement représentatif (mêmes OS, RSAT, modules, permissions, données).
- [ ] Comparer les sorties 5.1 / PS7 : objets, types, encodage, erreurs, codes de sortie, journaux, fichiers générés.
- [ ] Tester les scénarios d'échec, pas seulement les exécutions nominales.
- [ ] Valider les tâches planifiées après redémarrage et sous le compte de service prévu.
- [ ] Mesurer la latence des modules exécutés via `-UseWindowsPowerShell`.
- [ ] Définir un seuil d'acceptation : aucun module critique non migré, aucune régression de sortie, aucune dépendance 5.1 non documentée.

## Gabarit de décision

| Résultat d'audit | Décision |
|---|---|
| Module nativement compatible et tests concluants | Migrer vers PS7 |
| Module compatible mais comportement différent | Migrer avec tests de régression et correctifs |
| Module utilisable seulement via `-UseWindowsPowerShell` | Solution transitoire : documenter les limites de sérialisation et prévoir le remplacement |
| Module sur la deny list (ex. `PSScheduledJob`) | Rester en 5.1 ou refondre |
| Module retiré (ex. `ISE`, `PSWorkflow`) | Remplacer l'outil ou réécrire le flux |
| Script PS7 uniquement | Ajouter `#requires -Version 7` et appeler `pwsh.exe` explicitement |
| Script hybride 5.1 / PS7 | Éviter la syntaxe PS7 uniquement et valider les deux runtimes |

## Stratégie de migration recommandée

1. Inventorier scripts, tâches planifiées, services, modules tiers et dépendances COM / .NET Framework.
2. Identifier les blocages : modules compatibles 5.1 uniquement, workflows, jobs planifiés, outils GUI/ISE, appels à des API .NET Framework spécifiques.
3. Tester sous PowerShell 7 dans un environnement isolé, avec les mêmes entrées, permissions et sorties attendues.
4. Migrer d'abord les scripts autonomes, puis les automatisations critiques et les intégrations avec des modules Windows.
5. Conserver 5.1 uniquement pour les dépendances réellement incompatibles, plutôt que de le supprimer prématurément.

L'objectif raisonnable : faire de PowerShell 7 l'environnement par défaut pour les nouveaux scripts, l'administration et le développement, tout en gardant 5.1 comme filet de sécurité pour les composants ou modules qui ne sont pas encore prêts. La migration est une transition contrôlée, pas une bascule binaire. `-UseWindowsPowerShell` reste un pont temporaire.

## Limites et points à vérifier

Recoupement effectué le 24 novembre 2026 avec des pages Microsoft Learn, le blog de l'équipe PowerShell, le support Microsoft et la presse, complété par un test sur Windows 11.

**Confirmé**

- `Appx` : le tableau de compatibilité Microsoft le classe « Natively Compatible » sur Windows 10 / Server 1809+, avec la mention de passer par la couche de compatibilité avec PowerShell 7.1. Le seuil exact de version reste à tester sur l'environnement cible.
- Deny list par défaut de la couche de compatibilité : `PSScheduledJob`, `BestPractices`, `UpdateServices`.
- Comportement de `PSModulePath` lors d'un lancement de `powershell.exe` via un processus intermédiaire (risque de `CommandNotFoundException`).
- Côté PS7, `$?` d'un exécutable natif vaut vrai pour le code de sortie 0, faux sinon (Microsoft Learn).
- `$?` et stderr : test réalisé le 24 novembre 2026 sous Windows 11, dans une console, avec Windows PowerShell 5.1.26100.10068 et PowerShell 7.6.10, `cmd.exe` appelé par son chemin complet et `2>&1`.
  - Écriture sur stderr avec le code de sortie 0 : `$?` = False en 5.1, True en 7.
  - Code de sortie 3 : `$?` = False dans les deux éditions, `$LASTEXITCODE` = 3.
  - `(Write-Error)` : `$?` = True en 5.1, False en 7, ce qui est conforme à la documentation Microsoft.
  - Le test n'a pas couvert l'absence de redirection `2>&1`, les hôtes autres que la console (ISE, VS Code) ni d'autres versions.
- Côté 5.1, la ligne « Runtime » (.NET Framework v4.5) est confirmée par Microsoft Learn. La précision « .NET Standard 2.0 » côté PS7, présente dans l'export d'origine, a été remplacée par une formulation générique.

**Corrigé**

- `Microsoft.PowerShell.LocalAccounts` : indiqué « retiré » dans la première version de ce document. Il n'est plus livré avec PowerShell 7, mais le module fourni par Windows est classé « Natively Compatible » sur Windows 10 / Server 1809+.
- Une phrase du PDF source suggérait que `Start-Process notepad &` crée un job ; l'exemple a été remplacé par `Get-Process &`.
- Exemple `PSModulePath` : `cmd` remplacé par `cmd.exe`, car `cmd` seul peut désigner un autre fichier du `PATH`.

**À vérifier**

- Date de fin de support de PowerShell 7.6 LTS (14 novembre 2028) : issue du PDF source, non recoupée.
- Éléments MSIX (compte SYSTEM, DISM, paramètres machine) : sources secondaires uniquement.
- Forme exacte `en-us` de l'URL `differences-from-windows-powershell` : reprise du lien cité par Next, non ouverte directement.
- Nom et date de la prochaine version de Windows : estimation de la presse.

## Sources

- Next, article d'origine : <https://next.ink/269400/powershell-7-va-integrer-windows-les-entreprises-vont-devoir-verifier-sa-compatibilite/>
- The Register : <https://www.theregister.com/os-platforms/2026/11/18/powershell-7-heading-into-windows-as-51-hits-the-end-of-the-road/5310791>
- Microsoft Support, dépréciation de Windows PowerShell 5.1 (KB5137172) : <https://support.microsoft.com/en-us/servicing/os/windows/docs/2026/11/kb5137172-windows-powershell-5-1-deprecation>
- Blog PowerShell, « PowerShell 7 is coming to Windows » : <https://devblogs.microsoft.com/powershell/powershell-7-coming-to-windows/>
- Microsoft Learn, migration de 5.1 vers 7 : <https://learn.microsoft.com/powershell/scripting/install/migrating-from-windows-powershell-51-to-powershell-7>
- Microsoft Learn, différences avec Windows PowerShell : <https://learn.microsoft.com/en-us/powershell/scripting/whats-new/differences-from-windows-powershell?view=powershell-7.6>
- Microsoft Learn, compatibilité des modules : <https://learn.microsoft.com/en-us/powershell/windows/module-compatibility>
- Microsoft Learn, `about_Windows_PowerShell_Compatibility` : <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_windows_powershell_compatibility>
- Microsoft Learn, `about_PSModulePath` : <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_PSModulePath>
- Microsoft Learn, `about_Error_Handling` : <https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_error_handling>
- Techzine (MSIX, points ouverts) : <https://www.techzine.eu/news/applications/145897/powershell-7-will-be-included-by-default-in-the-next-windows-release/>
- BetaNews (annonce Microsoft sur le calendrier de 5.1) : <https://betanews.com/article/microsoft-deprecation-windows-powershell-5-1/>

## Licence

Ce dépôt est distribué sous licence MIT. Voir le fichier [LICENSE](LICENSE).
