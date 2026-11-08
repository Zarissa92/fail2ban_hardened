# f2b.sh — Blocage nginx « Tu es banni » via fail2ban

Script de déploiement Bash qui branche fail2ban directement sur nginx : quand une IP est bannie
par la jail `sans_pitie`, elle est servie avec une page **403 « Tu es banni »** — sans passer par
iptables/nftables. Le blocage est appliqué par nginx lui-même via une directive `geo`.

Testé sur **Debian 13 (trixie)** avec nginx et fail2ban (mawk 1.3.4).

## ⚙️ Principe de fonctionnement

```text
fail2ban (jail sans_pitie)
    └─ action nginx-geo-ban
        └─ echo "<ip> 1;" >> /etc/nginx/banned_ips.conf  +  reload nginx
            └─ geo $banned_ip (conf.d/banned.conf)
                └─ snippet fail2ban-banni.conf
                    └─ rewrite vers /banni → 403 « Tu es banni »
```

Avantages par rapport à un ban pare-feu classique :

- aucune règle iptables/nftables à maintenir ;
- la page de ban est personnalisable (HTML directement dans le snippet) ;
- les IP bannies restent bloquées même si fail2ban est arrêté (la liste est un fichier nginx).

## 📋 Prérequis

- Debian (ou dérivée) avec **nginx** et **fail2ban** installés ;
- une jail `[sans_pitie]` déjà définie dans `/etc/fail2ban/jail.d/hardening.local`
  (le script s'arrête sinon avec un message explicite) ;
- exécution en **root** (`sudo`).

## 🚀 Utilisation

```bash
sudo ./f2b.sh /etc/nginx/sites-available/default
```

L'argument est le fichier de site nginx à instrumenter. Le script est **idempotent** :
on peut le relancer sans risque après chaque évolution de la configuration.

### Test de bout en bout

```bash
# Bannir une IP de test (203.0.113.10 = TEST-NET-3, sûre)
sudo fail2ban-client set sans_pitie banip 203.0.113.10
grep 203.0.113.10 /etc/nginx/banned_ips.conf      # doit afficher : 203.0.113.10 1;
curl --resolve mon-domaine.tld:80:203.0.113.10 http://mon-domaine.tld   # → 403 « Tu es banni »

# Débannir
sudo fail2ban-client set sans_pitie unbanip 203.0.113.10
```

## 📁 Fichiers gérés

| Fichier | Rôle |
|---|---|
| `/etc/fail2ban/action.d/nginx-geo-ban.conf` | Action fail2ban : ajoute/retire l'IP et reload nginx |
| `/etc/nginx/conf.d/banned.conf` | Directive `geo $banned_ip` |
| `/etc/nginx/banned_ips.conf` | Liste des IP bannies (jamais écrasée par le script) |
| `/etc/nginx/snippets/fail2ban-banni.conf` | `if ($banned_ip)` + `location = /banni` (403 HTML) |
| `/etc/fail2ban/jail.d/hardening.local` | `banaction = nginx-geo-ban` injecté dans `[sans_pitie]` |
| fichier de site passé en argument | `include` du snippet inséré dans **tous** les blocs `server` |

## 🛡️ Sauvegarde et rollback

À chaque exécution, le script sauvegarde l'état précédent des fichiers gérés dans :

```text
/root/banni-backup-<AAAAMMJJ-HHMMSS>/
```

En cas d'erreur (via `set -Eeuo pipefail` + trap `ERR`) : restauration automatique de tous
les fichiers, validation nginx, reload de nginx et redémarrage de fail2ban.
Un échec du test `nginx -t` ou du redémarrage de fail2ban déclenche donc toujours un retour
à l'état antérieur — le serveur ne reste jamais dans un état cassé.

## 🔧 Détails techniques

- **Idempotence** : les includes `fail2ban-banni` existants sont purgés puis réinsérés ;
  le `banaction` est remplacé ou inséré après `[sans_pitie]` s'il est absent ;
  `banned_ips.conf` n'est **jamais** écrasé (il contient les bans en cours).
- **Pas de doublons** : `actionban` vérifie avec `grep -qF` avant d'ajouter une IP,
  y compris lorsque fail2ban restaure ses bans depuis sa base sqlite au redémarrage.
- **Persistance** : nginx lit `banned_ips.conf` à chaque reload indépendamment de fail2ban ;
  un arrêt prolongé de fail2ban ne débloque donc aucune IP.
- **Page de ban** : modifiable dans `/etc/nginx/snippets/fail2ban-banni.conf`, puis
  `sudo nginx -t && sudo systemctl reload nginx`.

## 🩺 Dépannage

| Symptôme | Cause probable / solution |
|---|---|
| `La jail sans_pitie est introuvable` | La section `[sans_pitie]` n'existe pas encore dans `hardening.local` — déployez d'abord la jail. |
| `Ligne banaction introuvable dans [sans_pitie]` | Le plus souvent : classes `[[:space:]]` corrompues lors d'un copier-coller (backslash parasite → `[[\:space:]]`). Vérifiez avec `grep -n 'space' f2b.sh \| cat -A` et corrigez avec `sed -i 's/\[\[\\:/[[:/g' f2b.sh`. |
| L'IP est bannie mais pas bloquée | `grep <ip> /etc/nginx/banned_ips.conf` ; vérifiez que `nginx -T \| grep banned` inclut bien le `geo` et le snippet dans le bon bloc `server`. |
| fail2ban ne redémarre pas | `journalctl -u fail2ban -n 20` — un `nginx -t` en échec dans l'action fait échouer le ban : corrigez la conf nginx d'abord. |

## 📜 Licence

Ce projet est publié sous **GNU General Public License v3.0 (GPL-3.0-or-later)**.

```text
f2b.sh — blocage nginx « Tu es banni » via fail2ban
Copyright (C) 2026  <votre nom>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <https://www.gnu.org/licenses/>.
```

Pensez aussi à ajouter l'en-tête SPDX en haut du script :

```bash
# SPDX-License-Identifier: GPL-3.0-or-later
```