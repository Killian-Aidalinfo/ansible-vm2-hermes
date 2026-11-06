# ansible-vm2-hermes

Socle d'une VM Ubuntu 24.04 (TP CESI) avant d'y installer Hermes et Hindsight.

| Rôle | Ce qu'il fait |
|---|---|
| `base` | `apt update` + `dist-upgrade` (reboot si besoin), outils, SSH durci (clé seule, pas de root), ufw (22/80/443) |
| `docker` | Docker CE + plugin compose depuis le dépôt officiel, utilisateur ajouté au groupe `docker` |
| `crowdsec` | moteur CrowdSec sur l'hôte (API sur `172.17.0.1:8080`), bouncer firewall nftables, scénario SSH strict (3 échecs/~1 h ⇒ ban 24 h), clé prête pour un futur bouncer Traefik |

## Lancer

```bash
ansible-galaxy collection install ansible.posix   # si absente
ansible-playbook site.yml
```

Adapter `inventory.ini` (hôte, utilisateur, clé SSH). Le playbook désactive l'authentification par mot de passe : vérifier que la connexion par clé fonctionne avant.

## À savoir

- Les ports publiés par Docker contournent ufw : exposer les services derrière un reverse proxy ou les binder sur `127.0.0.1`.
- Le bouncer firewall CrowdSec doit être installé avant de déplacer l'API locale sur `172.17.0.1`, sinon son script d'installation échoue (ordre déjà respecté dans le rôle).
