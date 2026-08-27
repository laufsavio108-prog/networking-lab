# networking-lab

Firewall de host com política **default deny** numa VM Kali, com a filtragem
comprovada por medição e o bloqueio confirmado em log.

O objetivo não era configurar um firewall — era praticar o ciclo inteiro:
definir a política, aplicá-la, **provar que ela bloqueia** e verificar que o
bloqueio **deixa rastro**. Configuração qualquer tutorial mostra. O que está
documentado aqui é a evidência.

```
Internet → Firewall (ufw) → Nginx (80) → App
```

## Política aplicada

| Regra | Porta | Origem | Por quê |
|---|---|---|---|
| deny incoming (padrão) | todas | — | O que não for liberado explicitamente fica fechado |
| allow outgoing (padrão) | todas | — | A máquina precisa de updates e DNS |
| allow | 80/tcp | qualquer | O site existe para ser acessado |
| allow | 443 | qualquer | Idem, versão criptografada |
| allow | 22/tcp | 192.168.56.1 | Administração apenas do host confiável |
| logging on | — | — | Bloqueio sem registro não é detectável |

Duas observações que fiz sobre a minha própria configuração:

- A regra da **443 está sem o sufixo `/tcp`**, ao contrário da 80. Sem ele, o
  ufw libera TCP e UDP. Como o nginx aqui não serve QUIC/HTTP3, o UDP na 443 é
  superfície aberta sem uso.
- O `ufw status` reporta `Logging: on`, mas **`/var/log/ufw.log` não existe**
  nesta Kali — sem `rsyslog` instalado, o registro vai para o journal do
  systemd. Quem seguisse um tutorial e não encontrasse o arquivo concluiria
  que o log não está funcionando. Está: `journalctl -k | grep 'UFW BLOCK'`.

## O que foi comprovado

Todas as saídas estão em [`evidence/2026-08-25/`](evidence/2026-08-25/), cada
arquivo com o comando que o gerou e a data.

| Evidência | O que prova |
|---|---|
| `01-politica-firewall.txt` | Firewall ativo, default deny na entrada, logging ligado |
| `02-portas-escutando.txt` | Quais processos de fato escutam — a superfície real da máquina |
| `03-http-responde.txt` | O caminho permitido funciona (`200 OK`, `Server: nginx`) |
| `06-tempo-filtragem.txt` | A porta sem regra é **filtrada**, não apenas fechada |
| `05-bloqueio-registrado.log` | A tentativa barrada **gerou registro** |

### A porta bloqueada não é só "fechada"

Medido do host (Windows, `192.168.56.1`) contra a VM:

| Porta | Regra | Tempo até resposta | Resultado |
|---|---|---|---|
| 80 | liberada | **0,0047 s** | conecta |
| 9999 | sem regra | **21,03 s** | timeout |

Mostrar que a porta 80 responde não prova nada: ela responderia sem firewall
nenhum. A prova está na diferença. Uma porta apenas **fechada** devolve RST em
milissegundos — *connection refused*, o host respondeu que ninguém escuta ali.
A espera de 21 segundos significa que **nada respondeu**: o pacote foi
descartado em silêncio antes de chegar ao sistema. Só isso comprova filtragem.

### O bloqueio virou registro

```
Aug 25 09:05:29 kali kernel: [UFW BLOCK] IN=eth1 SRC=192.168.56.1 DST=192.168.56.101 PROTO=TCP SPT=63299 DPT=9999 SYN
Aug 25 09:05:30 ...  SPT=63299 DPT=9999 SYN
Aug 25 09:05:32 ...  SPT=63299 DPT=9999 SYN
Aug 25 09:05:36 ...  SPT=63299 DPT=9999 SYN
Aug 25 09:05:44 ...  SPT=63299 DPT=9999 SYN
```

**São cinco linhas, mas uma única tentativa.** A porta de origem é a mesma nas
cinco (`SPT=63299`), o que significa a mesma conexão, e os intervalos dobram:
1, 2, 4 e 8 segundos. É a pilha TCP do cliente retransmitindo o SYN que nunca
recebeu resposta, com backoff exponencial, até desistir.

Isso importa na prática: uma regra de alerta do tipo *"5 bloqueios da mesma
origem = varredura"* dispararia aqui, e estaria errada. Varredura real aparece
como muitos `DPT` **diferentes** em sequência; retransmissão aparece como o
mesmo `DPT` com intervalos dobrando.

E as duas evidências se confirmam: do primeiro SYN (09:05:29) até a desistência
são ~21 segundos, o mesmo valor medido no host por caminho independente.

Dois detalhes que a própria linha entrega: `TTL=128` indica origem Windows
(Linux parte de 64, e o pacote não cruzou roteador), e o prefixo `08:00:27` no
campo `MAC=` é da Oracle — o log revela que é ambiente virtualizado.

## Endurecimento do SSH

A regra de firewall restringe a **origem** que alcança a porta 22. Ela não diz
nada sobre o que o `sshd` aceita de quem chega — são camadas independentes.
Conferindo a configuração efetiva (`sshd -T`, não o arquivo), encontrei três
exposições:

| Opção | Antes | Depois |
|---|---|---|
| `permitrootlogin` | `without-password` | `no` |
| `passwordauthentication` | `yes` | `no` |
| `x11forwarding` | `yes` | `no` |

A mais relevante é a segunda: eu já autenticava por chave, mas o servidor
continuava aceitando senha — a superfície de força bruta permanecia inteira. A
chave não fecha essa porta, só oferece um caminho melhor pela porta que segue
aberta.

Correção em [`hardening/99-hardening.conf`](hardening/99-hardening.conf), como
arquivo próprio em `sshd_config.d/`, sem tocar no `sshd_config` da distribuição:
atualização de pacote não conflita, o que é meu fica isolado, e reverter é
apagar um arquivo.

**Verificação**, com o cliente proibido de usar a chave:

```
$ ssh -o PubkeyAuthentication=no kali@192.168.56.101
kali@192.168.56.101: Permission denied (publickey).
```

Sem prompt de senha. Entre parênteses o servidor lista os métodos que ainda
aceita, e só restou `publickey`. O par de testes fecha o argumento: com a chave
entra, sem ela não entra de forma alguma.

O arquivo também define `KbdInteractiveAuthentication no`. **Essa não corrigiu
exposição** — já estava desligada antes da mudança. Mantive como configuração
defensiva explícita, porque em sistemas com `UsePAM yes` esse método pode
aceitar senha mesmo com `PasswordAuthentication no`. Registro aqui que nesta
máquina ela não mudou nada, para não contar como conquista o que não foi.

## Reproduzindo

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 192.168.56.1 to any port 22 proto tcp
sudo ufw logging on
sudo ufw enable
```

⚠️ Confirme que a regra da 22 existe **antes** de habilitar. Default deny sem
SSH liberado tranca o administrador fora da própria máquina, e num servidor
remoto isso significa perder o acesso.

Para o SSH, copie `hardening/99-hardening.conf` para
`/etc/ssh/sshd_config.d/`, valide com `sudo sshd -t` e só então
`sudo systemctl restart ssh`. Reiniciar com configuração inválida é como o
serviço não volta.

## Limitações assumidas

- **Saída irrestrita.** Prático para o lab, mas é justamente o caminho de
  exfiltração e de C2. Em ambiente real mereceria restrição própria.
- **Sem TLS real.** A 443 está liberada na política, mas o lab serve HTTP.
- **Firewall só de host.** Não há segmentação de rede nem firewall de borda.
- **VM local**, em rede host-only, sem exposição à internet.

## Notas de método

Durante a coleta, o primeiro `diff` do `sshd` saiu inválido: os filtros `grep`
do "antes" e do "depois" não eram idênticos, então o resultado misturava
mudança real com mudança de instrumento. Refiz as duas medições com o mesmo
filtro. Comparação antes/depois só vale se a medição for a mesma dos dois
lados.

---

Parte do meu roadmap rumo a Blue Team / SOC.
Outros labs: [linux-labs](https://github.com/laufsavio108-prog/linux-labs) ·
[secure-docker-stack](https://github.com/laufsavio108-prog/secure-docker-stack)
