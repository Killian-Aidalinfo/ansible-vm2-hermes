# ansible-vm2-hermes

VM Ubuntu 24.04 (TP CESI) : socle sécurisé, puis l'agent [Hermes](https://github.com/NousResearch/hermes-agent) avec sa mémoire long terme [Hindsight](https://github.com/vectorize-io/hindsight) auto-hébergée.

| Rôle | Ce qu'il fait |
|---|---|
| `base` | `apt update` + `dist-upgrade` (reboot si besoin), outils, SSH durci (clé seule, pas de root), ufw (22/80/443) |
| `docker` | Docker CE + plugin compose depuis le dépôt officiel, utilisateur ajouté au groupe `docker` |
| `crowdsec` | moteur CrowdSec sur l'hôte (API sur `172.17.0.1:8080`), bouncer firewall nftables, scénario SSH strict (3 échecs/~1 h ⇒ ban 24 h), clé prête pour un futur bouncer Traefik |
| `agents` | stack Compose dans `/opt/agents` : Hindsight 1.2.0 (embeddings et reranker locaux) + Postgres pgvector, Hermes v0.23.0 avec le plugin Hindsight en mode `local_external` |

## Lancer

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml
```

Secrets : tout est dans `credentials/` (hors git). Les mots de passe de Hindsight sont générés au premier lancement ; la clé LLM se met à la main :

```bash
printf '%s' 'sk-or-…' > credentials/llm_api_key   # fournisseur : llm_provider dans group_vars/all.yml
ansible-playbook site.yml
```

Sans clé LLM, Hindsight tourne sans extraction (`HINDSIGHT_API_LLM_PROVIDER=none`) et Hermes ne peut pas répondre.

## Brancher un bot de messagerie

Le token reste sur la VM, dans `~/.hermes/.env` (exemple Discord) :

```bash
cat >> ~/.hermes/.env <<'ENV'
DISCORD_BOT_TOKEN=…
DISCORD_ALLOWED_USERS=<ton ID Discord>
ENV
docker restart hermes
```

Variantes Telegram : `TELEGRAM_BOT_TOKEN` et `TELEGRAM_ALLOWED_USERS`.

## Accéder à Hindsight

Les ports n'écoutent que sur `127.0.0.1` : passer par un tunnel SSH.

```bash
ssh -L 9999:127.0.0.1:9999 -L 8888:127.0.0.1:8888 etudiant@<vm>
```

- Interface : http://localhost:9999 (clé : `credentials/hindsight_cp_key`)
- API / MCP : http://localhost:8888/mcp/hermes/ avec `Authorization: Bearer <credentials/hindsight_api_key>`

## Vérifier

```bash
docker compose -f /opt/agents/docker-compose.yml ps
docker exec hermes hermes memory status
docker exec hermes hermes doctor
curl -s http://127.0.0.1:8888/health
```

Adapter `inventory.ini` (hôte, utilisateur, clé SSH). Le playbook désactive l'authentification par mot de passe : vérifier que la connexion par clé fonctionne avant.

## À savoir

- Les ports publiés par Docker contournent ufw : la stack ne publie que sur `127.0.0.1`, garder cette règle pour tout nouveau service.
- Ne jamais lancer deux gateways Hermes sur le même `~/.hermes`, ni réutiliser un token de bot déjà actif ailleurs.
- Le bouncer firewall CrowdSec doit être installé avant de déplacer l'API locale sur `172.17.0.1`, sinon son script d'installation échoue (ordre déjà respecté dans le rôle).
