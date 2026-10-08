# Questions d'un RSSI

Les questions qu'une DSI ou un RSSI pose avant d'installer Registris, avec les
réponses telles qu'elles sont dans le code, et le fichier à ouvrir pour
vérifier. Rien ici n'est une promesse : tout se relit dans `src/` et se rejoue
avec `npm test`.

Contact : <contact@irreductix.fr>. Pour une faille, voir [SECURITY.md](../SECURITY.md).

## Qui est derrière, et que se passe-t-il si ça s'arrête ?

Registris est publié par [Irréductix](https://irreductix.fr), un collectif
informel qui construit des outils libres pour le service public. Ce n'est pas
une société : il n'y a ni abonnement, ni version payante, ni support
contractuel. La licence est l'EUPL-1.2, compatible avec les règles de
publication du secteur public.

Si le collectif s'arrête, vous gardez tout : le code (une cinquantaine de
fichiers dans `src/`, lisibles en une journée), les tests (plus de deux cents, qui
rejouent les parcours complets en HTTP), la nomenclature logicielle de chaque
version, et surtout vos données, dans un fichier SQLite et un dossier de pièces
que n'importe quel outil lit, exportables en CSV et en ZIP depuis l'écran. Il
n'y a aucun format propriétaire et aucune dépendance à un service en ligne.

## Qu'est-ce qui sort de mon réseau ?

Rien. L'application ne fait aucun appel sortant : pas de télémétrie, pas de
mise à jour automatique, pas de police ni de script chargé depuis un CDN. Les
seules connexions sont celles que vous configurez : votre annuaire en LDAPS et
votre relais SMTP. Vérifiable : `grep -rn "fetch(\|http.request\|https.request" src/`
ne trouve que des appels du navigateur vers l'application elle-même.

Le serveur écoute sur le port et l'adresse que vous fixez (`HOTE`, `PORT`),
`127.0.0.1` seulement si vous le placez derrière un reverse-proxy.

## Quelles données sont traitées ?

Des données d'agents : matricule, nom, prénom, courriel professionnel,
habilitations demandées et obtenues, pièces justificatives jointes (courriels,
captures, PDF), extractions de comptes déposées pour un rapprochement, et le
journal des actions avec l'adresse IP des connexions. **Aucune donnée patient.**

La fiche pour le registre des traitements, les durées de conservation, la purge
et l'information des agents sont dans [RGPD.md](RGPD.md).

## Où sont les secrets ?

Dans un seul fichier de configuration, hors du dossier de l'application :
`registris.env` (sous Windows, `C:\ProgramData\Registris\registris.env`, lisible
par le compte de service seulement). Il contient le mot de passe du compte de
service de l'annuaire, celui du relais SMTP, et le secret de session.

- Le secret de session est fourni par `SESSION_SECRET` ou généré au premier
  démarrage dans `session.secret`, à côté de la base, en droits `0600`
  ([src/config.js](../src/config.js)).
- Les mots de passe des comptes locaux sont hachés avec scrypt, sel aléatoire,
  comparaison à temps constant ([src/auth.js](../src/auth.js)). Les mots de
  passe de l'annuaire ne sont jamais stockés.
- Les jetons de l'API en lecture sont affichés une seule fois à leur création ;
  la base ne garde que leur empreinte SHA-256 ([src/api.js](../src/api.js)).
- La base ne contient donc aucun secret réutilisable.

## Comment on s'authentifie ?

Par votre Active Directory, en LDAPS : bind avec un compte de service puis
recherche de l'agent, ou bind direct de l'agent sans compte de service. Les
rôles viennent de quatre groupes de l'annuaire (administrateurs, référents,
contrôleurs, utilisateurs), groupes imbriqués en option. Le certificat de
l'autorité interne se déclare dans la configuration. Tout est détaillé dans
[DEPLOIEMENT.md, section 4](DEPLOIEMENT.md#4-active-directory), et
`registris verifier-config` teste le compte de service, la base de recherche
et chaque groupe avant la mise en service.

Les comptes locaux servent à démarrer, à tester, et de secours si l'annuaire
est injoignable ; en mode annuaire, c'est le seul cas où ils fonctionnent.

Protection de la connexion : blocage de dix minutes après cinq échecs, par
adresse IP et identifiant, persistant en base ; un annuaire injoignable est
distingué d'un mauvais mot de passe et n'alimente pas le compteur. Toutes les
tentatives, réussies ou non, sont au journal d'audit avec l'adresse IP.

Session : cookie `httpOnly`, `sameSite=lax`, `secure` dès que HTTPS est là,
régénérée à la connexion, expirée huit heures après la connexion, stockée en
base et purgée à l'expiration. Pas de SSO SAML ni OpenID Connect : si c'est un
prérequis chez vous, dites-le nous.

## Qui a le droit de faire quoi ?

Quatre rôles, vérifiés sur chaque route côté serveur ([src/roles.js](../src/roles.js)) :

| Rôle | Peut |
|---|---|
| utilisateur | demander un accès pour lui ou un collègue, signaler un départ, suivre ses demandes |
| référent | valider, exécuter, refuser, fermer, sur **ses applications seulement** |
| contrôleur | lire tout le registre, le journal, les rapprochements ; ne modifie rien |
| administrateur | catalogue, comptes, configuration, jetons d'API, campagnes de revue |

Un référent sans périmètre déclaré couvre toutes les applications : déclarez
les périmètres dès la mise en service. L'accord d'un cadre d'UF peut être exigé
avant le référent, par application. Une suppléance transfère un périmètre
pendant une absence, jusqu'à une date, sans reconnexion. L'API `/api/v1` est en
lecture seule, par jeton révocable.

## Et si quelqu'un vole la base ?

Il obtient les données d'agents citées plus haut, les extractions de comptes
déposées, et des empreintes inutilisables : hachés scrypt des mots de passe
locaux, SHA-256 des jetons d'API. Les pièces justificatives sont des fichiers
sur le disque, sous un nom neutre, en clair : le chiffrement au repos relève du
système (disque ou volume chiffré), comme pour vos autres applications.
Les sauvegardes contiennent la même chose : traitez-les avec le même soin.

## Et si un administrateur retouche la base ?

Chaque action est inscrite dans un journal chaîné : l'empreinte SHA-256 de
chaque entrée couvre l'entrée précédente ([src/audit.js](../src/audit.js)).
Modifier ou supprimer une ligne casse la chaîne. `registris verifier`, à
planifier chaque jour, contrôle toute la chaîne et toutes les pièces ; les
écrans contrôlent depuis le dernier contrôle complet.

Contre le remplacement de la base entière, `registris ancrer`, à planifier
chaque jour, dépose la tête de chaîne hors de la base, dans un fichier daté ou
un courriel :
conservez ces ancrages sur un autre disque, un autre service ou chez une autre
équipe. Ce n'est pas un horodatage qualifié ; c'est un témoin que l'on
contrôle. Les limites sont écrites dans [SECURITY.md](../SECURITY.md), section
« Points connus et choix assumés ».

## HTTPS ?

Deux façons : un reverse-proxy devant l'application, qui porte le certificat
(recommandé, [DEPLOIEMENT.md, section 3](DEPLOIEMENT.md#3-https-par-reverse-proxy)),
ou le TLS direct avec un certificat PFX ou PEM. L'installateur Windows génère
un certificat auto-signé pour démarrer ou prend le vôtre. HSTS est envoyé dès
que le cookie est sécurisé. Une écoute sur le réseau sans TLS déclenche un
avertissement explicite au démarrage et dans `registris verifier-config` ; le
refus de démarrer en production est réservé à l'absence de secret de session.

## Quelle surface d'attaque côté web ?

- Content-Security-Policy avec un nonce par réponse pour les scripts, aucune
  ressource externe, aucun gestionnaire d'événement en ligne ; `X-Frame-Options
  DENY`, `X-Content-Type-Options nosniff`, `Referrer-Policy`
  ([src/serveur.js](../src/serveur.js)).
- Jeton CSRF par session, vérifié sur chaque formulaire, y compris multipart ;
  un corps multipart adressé à une route qui n'en attend pas est refusé avant
  tout traitement.
- Téléversements : liste blanche d'extensions, signature binaire contrôlée
  (PDF, PNG, JPEG, MSG), taille maximale, nom stocké neutre.
- Exports CSV : toute cellule commençant par un signe de calcul est neutralisée,
  pour que le fichier ouvert sur le poste de l'auditeur n'exécute rien.
- Pas de limitation de débit hors connexion : les autres routes exigent une
  session et un jeton par formulaire ; un abus par un compte authentifié se voit
  au journal.
- `/sante` répond sans session et donne le numéro de version, pour votre
  supervision : ne l'exposez pas hors du réseau interne.

## Dépendances et chaîne d'approvisionnement ?

Sept dépendances d'exécution (Express, express-session, multer, archiver,
nodemailer, ldapts, ldap-authentication), listées dans `package.json`, figées
dans `package-lock.json`. À chaque version, l'intégration continue :

- exécute `npm audit` sur les dépendances de production et échoue au niveau
  « high » ;
- construit l'archive, l'installateur Windows et l'image Docker **à partir du
  tag signé**, jamais depuis un poste ;
- publie une nomenclature logicielle CycloneDX (`registris-x.y.z.cdx.json`) et
  les sommes `SHA256SUMS` ;
- vérifie le Node.js embarqué pour Windows contre les sommes publiées par
  nodejs.org, et le lanceur de service WinSW par empreinte.

Dependabot ouvre une demande à chaque avis de sécurité sur une dépendance.
L'installateur Windows n'est pas signé par un certificat d'éditeur : SmartScreen
avertit, et c'est la somme SHA-256 publiée avec la release qui fait foi. L'image
Docker s'exécute sans privilège.

## Comment on met à jour ?

Sous Windows, on relance l'installateur de la nouvelle version : il arrête le
service, remplace l'application, garde les données et la configuration,
redémarre et contrôle. Ailleurs, `git pull`, `npm install --omit=dev`,
redémarrage du service. Le schéma de base se met à jour seul, par ajouts
seulement ; chaque version est testée en ouvrant une base de la version
précédente (voir les revues de version dans [SECURITY.md](../SECURITY.md)).
Les changements sont dans [CHANGELOG.md](../CHANGELOG.md), les versions sont
des tags signés.

## Sauvegardes et restauration ?

`registris sauvegarder` produit une archive avec un manifeste d'empreintes ;
`registris restaurer` refuse une archive dont un fichier ne correspond pas et
se contrôle sans rien écrire. À sauvegarder : la base, le dossier des pièces,
le fichier de configuration. Les ancrages de la chaîne vont ailleurs.
[DEPLOIEMENT.md, section 7](DEPLOIEMENT.md#7-sauvegardes-et-restauration).

## Journaux et supervision ?

- Le journal d'audit, dans l'application : connexions, créations, validations,
  consultations et dépôts de pièces, exports, avec l'acteur, l'horodatage et
  l'adresse IP des connexions. Exportable, chaîné, non purgé par la
  conservation : c'est la preuve.
- La sortie du service (erreurs d'annuaire, d'envoi de courriel, d'entretien)
  dans `journaux/` sous Windows, dans journald sous Linux.
- Pas de journal d'accès HTTP dans l'application : celui de votre reverse-proxy
  le fournit.
- `GET /sante` pour la supervision, `registris verifier-config` pour contrôler
  annuaire, messagerie, certificat et dossiers à tout moment, aussi depuis la
  page d'administration.

## Conservation et purge ?

`CONSERVATION_ANNEES` fixe la durée après la clôture du dernier accès d'un
agent parti ; `registris purger` efface son identité et supprime ses pièces,
avec un mode simulation et un bilan au journal. Le journal d'audit n'est pas
purgé. [RGPD.md](RGPD.md).

## Ce qui n'est pas couvert, pour ne pas le découvrir après

- Pas de SSO SAML ni OpenID Connect : Active Directory en LDAPS, ou comptes locaux.
- Pas de chiffrement applicatif au repos : c'est celui du système.
- Pas d'horodatage qualifié : l'ancrage est un témoin que vous conservez.
- Pas de limitation de débit au-delà de la connexion.
- Installateur Windows non signé.
- Pas d'audit de sécurité par un tiers à ce jour : le code est ouvert à cette
  fin, et un rapport serait publié ici.
- Pas de déclaration d'accessibilité certifiée ; ce qui est vérifié est dans
  [ACCESSIBILITE.md](ACCESSIBILITE.md).

## Comment l'auditer soi-même, en une heure

1. `git clone`, `npm ci`, `npm test` : les parcours complets se rejouent en HTTP
   contre une base temporaire.
2. `npm audit --omit=dev` et la nomenclature CycloneDX de la release.
3. Lire `src/serveur.js` (en-têtes, CSRF, routes multipart), `src/auth.js`,
   `src/roles.js`, `src/audit.js` : c'est là que tout se joue.
4. Installer sur une machine d'essai et lancer `registris verifier-config`.
5. Nous écrire ce qui manque : <contact@irreductix.fr>.
