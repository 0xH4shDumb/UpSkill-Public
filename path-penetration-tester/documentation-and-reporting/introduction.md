# Documentation et reporting

La competence technique ne suffit pas en pentest. Sans documentation rigoureuse et sans rapport de qualite, le travail d'exploitation n'a aucune valeur pour le client. Le rapport est le livrable principal d'un engagement, et c'est souvent la seule chose que le client conserve. Cette page explique pourquoi la documentation est aussi critique que l'exploitation elle-meme.

## Pourquoi

Un rapport bien redige transforme des heures de tests en recommandations actionnables. Un rapport mediocre transforme les memes heures en un document que personne ne lira. La documentation protege aussi le pentester en cas de litige, d'incident reseau ou de question du client sur les activites realisees.

## Comment ca marche

### Le rapport comme instantane

Un test d'intrusion capture l'etat de la securite d'un environnement a un instant donne. Le rapport doit le mentionner explicitement : dates de test, adresses IP source, perimetre couvert, et le fait que toute modification de l'environnement apres la periode de test echappe a l'evaluation.

### Scenarios ou la documentation sauve la mise

La documentation ne sert pas qu'a produire un joli PDF. Voici des situations reelles ou elle fait la difference.

{% tabs %}
{% tab title="VM detruite" %}
Lors d'un engagement de plusieurs semaines, la VM de test devient inutilisable (corruption du systeme de fichiers). Si les notes sont prises en local sur la VM sans sauvegarde externe, des semaines de travail sont perdues. Avec des notes synchronisees sur un stockage externe et des sauvegardes quotidiennes, la reconstruction de la VM prend quelques heures au lieu de recommencer l'engagement.
{% endtab %}
{% tab title="Accusation de crash reseau" %}
Le client accuse le pentester d'avoir fait tomber des serveurs critiques. Les logs de scan horodates, les fichiers de scope signes et les sorties brutes de Nmap permettent de prouver que les hotes affectes etaient bien dans le perimetre valide, ou au contraire que l'incident a une autre cause. Sans cette documentation, la responsabilite retombe sur le pentester par defaut.
{% endtab %}
{% tab title="Contestation des resultats" %}
Un administrateur reseau conteste les conclusions du rapport, affirmant que les scans ont perturbe le reseau. Les logs detailles et la sortie des outils prouvent que les scans ont suivi les bonnes pratiques et que le probleme venait d'une configuration de debug activee sur les equipements reseau.
{% endtab %}
{% endtabs %}

### Principes fondamentaux

| Principe | Explication |
|---|---|
| **Documenter en temps reel** | Prendre les notes pendant le test, pas apres. La memoire deforme les etapes |
| **Sauvegarder regulierement** | Ne jamais stocker toutes les donnees sur une seule VM. Synchroniser vers un stockage externe |
| **Tout horodater** | Chaque commande, chaque scan, chaque tentative d'exploitation doit porter un timestamp |
| **Conserver les sorties brutes** | Les logs des outils sont la preuve irrefutable de ce qui a ete fait |
| **Adapter au public** | Un rapport s'adresse a la fois a la direction (synthese) et aux equipes techniques (details) |

{% hint style="warning" %}
Le rapport n'est pas un exercice academique. Le client paie pour un livrable actionnable. Chaque page, chaque capture d'ecran, chaque recommandation doit avoir une raison d'etre dans le document.
{% endhint %}

## Pieges et galeres

- **Reporter la redaction a la fin** : ecrire le rapport le dernier jour de l'engagement produit un document bacle. Commencer la redaction des le premier jour, en remplissant les sections au fur et a mesure
- **Notes illisibles** : des notes prises a la va-vite sans structure ni contexte sont inutilisables trois jours plus tard. Prendre 30 secondes de plus pour decrire ce qu'on fait et pourquoi
- **VM unique sans sauvegarde** : une VM qui lache emporte tout le travail. Sauvegarder les notes et les preuves sur un support externe a la fin de chaque journee
- **Confusion entre draft et rapport final** : le client peut recevoir un draft pour commentaires, puis un rapport final integrant ses retours. Ne jamais envoyer un draft en tant que livrable definitif

## Memo express

| Element | A retenir |
|---|---|
| **Livrable principal** | Le rapport est le produit final du pentest |
| **Instantane** | Le rapport couvre une periode precise, pas un etat permanent |
| **Notes en temps reel** | Documenter pendant le test, pas apres |
| **Sauvegarde** | Ne jamais stocker uniquement sur la VM de test |
| **Double audience** | Direction (synthese) + technique (details) |
| **Horodatage** | Chaque action doit etre datee pour la tracabilite |

***
