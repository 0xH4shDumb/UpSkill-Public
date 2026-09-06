# Elevation de privileges Linux

L'elevation de privileges est l'etape qui transforme un acces initial limite en controle total du systeme. Sur Linux, les vecteurs sont nombreux : permissions speciales, configurations sudo trop permissives, services mal securises, conteneurs exploitables, vulnerabilites noyau. Ce module couvre l'ensemble du spectre, de l'enumeration initiale au durcissement.

## Pages

* [Introduction](introduction.md)
* [Enumeration du systeme](enumeration.md)
* [Permissions speciales : SUID, SGID et capabilities](permissions-speciales.md)
* [Sudo et groupes privilegies](sudo-et-groupes.md)
* [Abus de l'environnement](abus-environnement.md)
* [Services, cron jobs et techniques diverses](services-et-cron.md)
* [Conteneurs : Docker, Kubernetes et LXC](conteneurs.md)
* [Mecanismes internes : noyau, bibliotheques et hijacking](mecanismes-internes.md)
* [Vulnerabilites recentes (CVE)](vulnerabilites-recentes.md)
* [Durcissement Linux](durcissement.md)
