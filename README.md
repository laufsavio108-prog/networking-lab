# networking-lab

Cadeia Internet → Firewall → Nginx → App numa VM Kali,
com o princípio "bloquear tudo que não for necessário" (default deny).

## Arquitetura
Internet → Firewall (ufw) → Nginx (porta 80) → App

## Regras do firewall e o porquê

| Regra | Porta | Origem | Por quê |
|-------|-------|--------|---------|
| deny incoming (padrão) | todas | — | Default deny: o que não for liberado, fica fechado |
| allow outgoing (padrão) | todas | — | A máquina precisa acessar a internet (updates, DNS) |
| allow http | 80 | Anywhere | Servir o site é público por natureza |
| allow https | 443 | Anywhere | Idem, versão criptografada |
| allow ssh restrito | 22 | 192.168.56.1 | Administração só da máquina confiável (Windows) |

## Verificação
- `sudo ufw status numbered` → confere as regras ativas
- `curl -I 192.168.56.101` → Server: nginx (cadeia funcionando)
- `sudo ss -tlnp` → quem está escutando em cada porta
