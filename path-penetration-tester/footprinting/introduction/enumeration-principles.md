# Principes d'enumeration

L'enumeration est le processus de collecte d'informations sur une cible, que ce soit par des moyens actifs (scans, requetes directes) ou passifs (sources tierces, donnees publiques). C'est une boucle iterative : chaque element decouvert ouvre de nouvelles pistes a explorer, qui elles-memes menent a d'autres decouvertes.

## Pourquoi

Quand on demarre un test d'intrusion, la tentation est forte de foncer sur les premiers services trouves et de tenter un brute-force ou un exploit connu. En pratique, cette approche est contre-productive. Un brute-force sur SSH ou RDP genere du bruit, declenche les mecanismes de defense, et peut conduire au blacklisting de l'adresse source. C'est la fin du test avant meme d'avoir commence serieusement.

L'objectif du footprinting n'est pas de penetrer les systemes le plus vite possible. C'est de **cartographier l'ensemble des chemins qui y menent**. Comprendre comment l'infrastructure est construite, quels services sont exposes, quelles technologies sont en place, et ou se trouvent les points faibles potentiels.

{% hint style="info" %}
L'OSINT (Open Source Intelligence) repose exclusivement sur la collecte passive. L'enumeration, elle, combine actif et passif. Les deux sont complementaires, mais il faut les distinguer dans la methodologie pour ne pas melanger les phases.
{% endhint %}

## Comment ca marche

L'enumeration repose sur un ensemble de questions qui guident la collecte a chaque etape. On a tendance a se concentrer sur ce qui est visible, mais ce qu'on ne voit pas est souvent tout aussi revelateur.

Les questions fondamentales :

- Qu'est-ce qu'on observe ?
- Pourquoi est-ce visible ?
- Quelle image ca dessine de la cible ?
- Qu'est-ce qu'on peut en tirer ?
- Comment l'exploiter ?
- Qu'est-ce qu'on ne voit **pas** ?
- Pourquoi c'est cache ou absent ?
- Qu'est-ce que cette absence nous apprend ?

Ces questions s'appliquent a toutes les situations, quel que soit le contexte technique. Elles forcent a prendre du recul et a considerer l'infrastructure sous tous les angles, pas seulement celui des ports ouverts.

### Les trois principes

| # | Principe |
|---|---|
| 1 | Il y a toujours plus que ce qui est visible au premier regard. Considerer tous les points de vue. |
| 2 | Faire la distinction entre ce qu'on observe et ce qu'on n'observe pas. |
| 3 | Il existe toujours un moyen d'obtenir plus d'informations. Comprendre la cible en profondeur. |

{% hint style="success" %}
Quand on bloque sur un test, c'est rarement un manque de competences techniques. C'est presque toujours un manque de comprehension de la cible. Revenir aux principes d'enumeration debloque souvent la situation.
{% endhint %}

## Retour terrain

En mission, on constate que les testeurs qui echouent sont rarement ceux qui manquent d'outils ou de techniques. Ce sont ceux qui sautent la phase de comprehension pour passer directement a l'exploitation. Un pentester qui prend le temps de cartographier l'infrastructure, de comprendre les flux metier et les interactions entre services, finit par trouver des chemins que le brute-force n'aurait jamais revele.

L'analogie est celle du chasseur de tresor : il ne prend pas une pelle pour creuser au hasard. Il etudie les cartes, analyse le terrain, prepare son equipement. C'est exactement la meme logique en test d'intrusion.

***
