# Ecriture et import de modules

Metasploit est un framework communautaire. Des milliers de modules sont integres, mais il arrive regulierement qu'un exploit recent ne soit pas encore dans la base officielle, ou qu'on ait besoin d'adapter un PoC public pour l'integrer a son workflow. Savoir importer un module externe ou en ecrire un soi-meme est une competence qui fait la difference.

## Pourquoi

Pendant un pentest, on decouvre une vulnerabilite pour laquelle un PoC Python existe sur ExploitDB, mais pas de module Metasploit. Deux options : executer le PoC a la main, ou l'integrer dans Metasploit pour profiter de la gestion des sessions, de la base de donnees, et du pivoting. La seconde option est souvent plus efficace sur un engagement complexe.

## Comment ca marche

### Ou sont stockes les modules

```
/usr/share/metasploit-framework/modules/     # modules officiels
~/.msf4/modules/                              # modules personnels (prioritaire)
```

L'arborescence suit la convention de nommage de Metasploit.

```
modules/
|-- exploits/
|   |-- windows/
|   |-- linux/
|   |-- multi/
|-- auxiliary/
|   |-- scanner/
|   |-- gather/
|-- post/
|   |-- windows/
|   |-- linux/
```

{% hint style="info" %}
Les modules places dans `~/.msf4/modules/` sont charges au demarrage de msfconsole et sont prioritaires sur les modules systeme. C'est l'emplacement recommande pour les modules personnalises, car une mise a jour de Metasploit ne les ecrasera pas.
{% endhint %}

### Trouver des modules externes

**ExploitDB** est la source principale. Le filtre "Metasploit Framework (MSF)" permet de ne garder que les scripts Ruby compatibles.

```bash
# - Rechercher un module via searchsploit
searchsploit -t nagios3 --exclude=".py"

# - Les fichiers .rb sont compatibles Metasploit
# - Les fichiers .py necessitent un portage en Ruby
```

## En pratique

### Importer un module externe

```bash
# 1 - Telecharger le module depuis ExploitDB
searchsploit -m 9861

# 2 - Le copier dans le repertoire personnel
mkdir -p ~/.msf4/modules/exploits/unix/webapp/
cp 9861.rb ~/.msf4/modules/exploits/unix/webapp/nagios3_command_injection.rb

# 3 - Charger dans msfconsole
msf6 > reload_all
[*] Reloading modules from all module paths...

# 4 - Verifier que le module est disponible
msf6 > search nagios3_command_injection
msf6 > use exploit/unix/webapp/nagios3_command_injection
msf6 > info
msf6 > show options
```

{% hint style="warning" %}
Toujours lire le code Ruby d'un module externe avant de l'executer. Un module malveillant pourrait compromettre la machine de l'attaquant. Verifier les imports, les appels systeme, et les URL de callback.
{% endhint %}

### Structure d'un module Metasploit

Tous les modules Ruby suivent la meme structure de base.

```ruby
class MetasploitModule < Msf::Exploit::Remote
  Rank = ExcellentRanking

  include Msf::Exploit::Remote::HttpClient

  def initialize(info = {})
    super(update_info(info,
      'Name'           => 'Nom de l exploit',
      'Description'    => %q{Description de la vulnerabilite...},
      'Author'         => ['auteur'],
      'References'     => [['CVE', '2021-XXXXX']],
      'Platform'       => 'linux',
      'Arch'           => ARCH_X64,
      'Targets'        => [['Automatic', {}]],
      'DisclosureDate' => '2021-01-01',
      'DefaultTarget'  => 0
    ))

    register_options([
      OptString.new('TARGETURI', [true, 'Base path', '/']),
    ])
  end

  def check
    # - Verification non-intrusive de la vulnerabilite
  end

  def exploit
    # - Code d'exploitation
  end
end
```

### Composants cles

| Element | Role |
|---|---|
| `Rank` | Fiabilite du module (`ExcellentRanking`, `GreatRanking`, `NormalRanking`) |
| `include` (Mixins) | Fonctionnalites importees (HTTP, SMB, FTP, etc.) |
| `register_options` | Options configurables par l'utilisateur |
| `check` | Methode de verification (non-intrusive) |
| `exploit` | Code d'exploitation principal |

### Mixins courants

| Mixin | Fonctionnalite |
|---|---|
| `Msf::Exploit::Remote::HttpClient` | Requetes HTTP vers la cible |
| `Msf::Exploit::Remote::Tcp` | Connexions TCP brutes |
| `Msf::Exploit::PhpEXE` | Generation de payloads PHP |
| `Msf::Exploit::FileDropper` | Depot et nettoyage de fichiers sur la cible |
| `Msf::Auxiliary::Report` | Ecriture dans la base de donnees Metasploit |
| `Msf::Auxiliary::Scanner` | Scan multi-cibles avec RHOSTS |

### Porter un PoC Python en module Ruby

La demarche generale pour convertir un PoC Python en module Metasploit.

1. **Analyser le PoC** : comprendre la vulnerabilite, les requetes envoyees, les conditions d'exploitation
2. **Partir d'un module existant** : trouver un module similaire dans Metasploit et l'utiliser comme squelette
3. **Adapter la logique** : traduire les appels Python (`requests.get`) en appels Ruby (mixin `HttpClient`)
4. **Ajouter les metadonnees** : CVE, description, auteur, targets
5. **Implementer `check`** : methode de verification non-intrusive
6. **Tester** : dans un lab isole, verifier que le module fonctionne et que la session s'ouvre correctement

```bash
# - Conventions de nommage
# Fichier : snake_case, caracteres alphanumeriques et underscores uniquement
# Exemple : bludit_bruteforce_bypass.rb
```

{% hint style="success" %}
S'inspirer d'un module existant du meme type est la methode la plus efficace. La commande `info` sur un module similaire donne sa localisation sur le disque, ce qui permet de lire son code source directement.
{% endhint %}

### Mettre a jour les modules

```bash
# - Mise a jour complete du framework
sudo msfupdate

# - Recharger les modules apres une modification
msf6 > reload_all
```

## Pieges et galeres

- **Erreur de syntaxe Ruby** : une erreur dans le module empechera son chargement. Utiliser `ruby -c module.rb` pour verifier la syntaxe avant de le placer dans le repertoire
- **Mixin manquant** : oublier un `include` provoque des erreurs a l'execution (ex: `send_request_cgi` n'existe pas sans `HttpClient`)
- **Permissions** : les modules dans `/usr/share/metasploit-framework/` necessitent les droits root pour etre modifies. Preferer `~/.msf4/modules/`
- **Module non trouve apres import** : verifier que le chemin respecte l'arborescence (`exploits/os/service/nom.rb`) et lancer `reload_all`

## Retour terrain

En pentest, l'import de modules est plus frequent que l'ecriture. Quand un CVE recent sort avec un PoC Metasploit sur ExploitDB, pouvoir l'integrer en quelques minutes dans son msfconsole est un vrai gain de temps. L'ecriture de modules from scratch est plus rare, mais elle devient necessaire quand on decouvre une vulnerabilite pour laquelle aucun PoC public n'existe. Dans ce cas, partir d'un squelette existant et adapter la logique d'exploitation est la methode la plus realiste.

## Memo express

| Commande | Usage |
|---|---|
| `searchsploit -m <id>` | Telecharger un exploit depuis ExploitDB |
| `~/.msf4/modules/` | Repertoire des modules personnels |
| `reload_all` | Recharger tous les modules |
| `info` | Details et localisation du module |
| `ruby -c module.rb` | Verifier la syntaxe Ruby |
| `loadpath <chemin>` | Charger un repertoire de modules |

***
