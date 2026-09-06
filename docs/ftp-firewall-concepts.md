# Comprendre les règles UFW pour FTP (Control + Passive Mode)

```bash
sudo ufw allow 20:21/tcp       # FTP control
sudo ufw allow 40000:50000/tcp # FTP passive range (must match vsftpd config)
```

Ce document explique **tous les concepts théoriques** nécessaires pour comprendre pourquoi ces deux règles existent et comment elles fonctionnent ensemble.

---

## 1. Le protocole FTP utilise DEUX canaux

Contrairement à HTTP (un seul port, 80/443), FTP sépare la communication en deux connexions distinctes :

| Canal | Rôle | Port par défaut |
|---|---|---|
| **Control channel** (commandes) | Envoie les commandes (`USER`, `PASS`, `LIST`, `RETR`, `STOR`...) et reçoit les réponses du serveur | **21** |
| **Data channel** (données) | Transfère réellement les fichiers et les listings de dossiers | Dynamique (négocié) |

Le port 20 est historiquement associé au canal de données en mode **actif** (voir plus bas), ce qui explique la plage `20:21` dans la première règle.

---

## 2. Mode actif vs mode passif

C'est le concept le plus important à maîtriser.

### Mode actif (FTP Active)
- Le **client** ouvre le canal de contrôle vers le port 21 du serveur.
- Pour le transfert de données, c'est le **serveur** qui initie une connexion *vers* le client (depuis son port 20).
- Problème : si le client est derrière un NAT/firewall (quasi toujours le cas aujourd'hui), le serveur ne peut pas initier de connexion entrante vers lui → **échec du transfert**.

### Mode passif (FTP Passive) — celui utilisé ici
- Le client ouvre le canal de contrôle vers le port 21.
- Le client demande ensuite au serveur : *"donne-moi un port pour les données"* (commande `PASV`).
- Le serveur répond avec un port aléatoire choisi dans une **plage prédéfinie**.
- Le client initie **lui-même** la connexion de données vers ce port.
- Avantage : fonctionne bien mieux avec les NAT/firewalls modernes, car c'est toujours le client qui initie les connexions.

C'est pourquoi la configuration passive est aujourd'hui la norme, et pourquoi il faut ouvrir une **plage de ports** côté serveur (`40000:50000` dans l'exemple).

---

## 3. Pourquoi une plage de ports (`40000:50000`) et pas un seul port ?

- Chaque connexion de données passive a besoin de **son propre port**, choisi aléatoirement par le serveur dans la plage configurée.
- Si plusieurs clients se connectent en même temps (ou si un même client fait plusieurs transferts), chacun utilise un port différent de la plage.
- Cette plage est définie côté serveur FTP (ex: `vsftpd`) via des paramètres comme :
  ```
  pasv_min_port=40000
  pasv_max_port=50000
  ```
- **Le firewall (ufw) doit autoriser exactement la même plage**, sinon le serveur choisira un port pour la connexion de données que le firewall bloquera → transfert qui "se bloque" après l'authentification (symptôme classique : on peut se connecter et voir `login successful`, mais `LIST` ou le transfert de fichier reste bloqué).

---

## 4. UFW (Uncomplicated Firewall) — les bases nécessaires

- UFW est une surcouche simplifiée d'`iptables` sous Linux (Ubuntu/Debian).
- `sudo ufw allow PORT/tcp` ajoute une règle autorisant le trafic **entrant** sur ce port en TCP.
- La syntaxe `20:21/tcp` signifie *"autoriser tous les ports de 20 à 21 inclus, en TCP"*.
- FTP fonctionne exclusivement en TCP (jamais UDP), d'où le suffixe `/tcp`.
- Sans ces règles explicites, UFW bloque par défaut tout trafic entrant non autorisé — donc même si `vsftpd` tourne correctement, les connexions échoueront tant que le firewall ne laisse pas passer les bons ports.

---

## 5. Le lien entre firewall et configuration du serveur FTP

C'est le point critique mentionné dans le commentaire du code : **"must match vsftpd config"**.

Il y a une chaîne de cohérence à respecter :

```
Configuration vsftpd (pasv_min_port / pasv_max_port)
              ⇅  (doivent être identiques)
Règle UFW (plage de ports autorisée)
```

Si ces deux valeurs ne correspondent pas :
- Le serveur peut choisir un port passif que le firewall bloque.
- Résultat : le client se connecte, s'authentifie, mais les transferts de fichiers échouent ou "timeout".

---

## 6. Concepts complémentaires utiles

- **NAT (Network Address Translation)** : traduction d'adresses IP privées/publiques, source du problème que le mode passif résout.
- **TCP vs UDP** : FTP est basé sur TCP car il nécessite une transmission fiable et ordonnée (contrairement à UDP).
- **FTPS / SFTP** : alternatives sécurisées à FTP (FTPS = FTP + TLS ; SFTP = protocole différent basé sur SSH). Utile de connaître la différence si la sécurité du transfert est une préoccupation.
- **Ports privilégiés (<1024)** : le port 21 fait partie des ports réservés nécessitant des droits root pour être utilisés par un service, d'où l'usage de `sudo`.

---

## 7. Résumé visuel du flux passif complet

```
Client                                Serveur FTP
  |--- connexion TCP port 21 -------->|   (control channel)
  |<-- "220 Service ready" -----------|
  |--- USER / PASS ------------------>|
  |--- commande PASV ----------------->|
  |<-- "227 Entering Passive Mode      |
  |     (ip, port choisi dans 40000-50000)"
  |--- nouvelle connexion TCP vers    |
  |    le port indiqué --------------->|   (data channel)
  |<-- transfert de fichier/listing ---|
```

---

## À retenir

1. FTP = 2 canaux (contrôle + données), pas un seul comme HTTP.
2. Le mode passif est préféré aujourd'hui car compatible NAT/firewall côté client.
3. La plage de ports passifs doit être **identique** entre `vsftpd` et `ufw`.
4. Sans règle UFW correspondante, le transfert de données échoue même si l'authentification réussit.
