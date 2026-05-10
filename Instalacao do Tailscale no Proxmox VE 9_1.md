# Writeup — Configuração do Proxmox Homelab

**Data:** Maio de 2026  
**Objetivo:** Transformar um notebook antigo em um servidor Proxmox para hospedar aplicações pessoais, acessível remotamente via rede.

---

## 1. Contexto e Arquitetura Decidida

### Objetivo
Hospedar aplicações pessoais (controle de finanças, kanban, etc.) em um notebook antigo rodando Proxmox, acessível remotamente do computador principal.

### Arquitetura escolhida

```
Proxmox
└── LXC: "apps"
    └── Docker
        ├── app-financas
        ├── app-kanban
        ├── postgres (banco compartilhado)
        └── nginx (reverse proxy)
```

### Por que LXC + Docker e não VMs?
- O notebook tem recursos limitados — LXC é muito mais leve que VMs (não carrega um kernel inteiro)
- Docker dentro do LXC facilita o deploy de apps com `docker-compose`
- VMs ficam reservadas para o computador principal (32GB RAM), onde serão usadas para labs de segurança

### Stack de desenvolvimento escolhida
| Camada | Tecnologia |
|---|---|
| Backend | FastAPI (Python) |
| Frontend | React |
| Banco | PostgreSQL |
| Infra | docker-compose |
| Reverse proxy | Nginx |

---

## 2. Problema — Acesso à Interface Web do Proxmox

### Sintoma
Tentativa de acessar `https://192.168.15.25:8006` no navegador falhando sem resposta.

### Diagnóstico

**Verificar IP do Proxmox:**
```bash
ip a | grep inet
```

Output relevante:
```
inet 192.168.15.25/24 brd 192.168.15.255 scope global dynamic wlp1s0
inet 192.168.15.2/24 scope global vmbr0
```

- `192.168.15.25` → interface WiFi (`wlp1s0`) — **IP correto para acesso**
- `192.168.15.2` → bridge do Proxmox (`vmbr0`)

**Verificar se o serviço está rodando:**
```bash
systemctl status pveproxy
# Resultado: active (running)
```

**Verificar se a porta está escutando:**
```bash
ss -tlnp | grep 8006
# Resultado: LISTEN 0 4096 *:8006 *:*
```

**Verificar firewall:**
```bash
pve-firewall status
# Resultado: disabled/running — não era o problema
```

### Causa Raiz
O computador principal estava conectado via **cabo ethernet** enquanto o Proxmox estava no **WiFi**. O roteador (Vivo — MitraStar GPT-2742GX4X5v6-SV) possuía a função **"Private Wi-Fi Network"** habilitada, que isola dispositivos WiFi de dispositivos cabeados (e entre si).

**Confirmado com:**
```bash
# No computador principal
ping 192.168.15.25
# Resultado: timeout — sem comunicação
```

### Tentativa de Solução — Desabilitar isolamento no roteador

Acessado o painel do roteador em `http://192.168.15.1`.

Localização da configuração:
```
Wireless → Advanced 2.4GHz / Advanced 5GHz
→ Private Wi-Fi Network: Enabled → Disabled
```

**Resultado:** O WiFi do notebook caiu após salvar a configuração. O roteador estava reiniciando, mas o WiFi não voltou normalmente. A opção foi revertida.

> **Observação:** "Private Wi-Fi Network" é o nome que a Vivo/MitraStar usa para o **AP Isolation**. Desabilitar essa opção torna cinza outras configurações que dependem dela — isso é comportamento esperado.

### Solução Adotada — Tailscale (VPN Mesh)

Em vez de depender da configuração do roteador, a solução escolhida foi usar o **Tailscale**, uma VPN mesh que conecta dispositivos independente de isolamento de rede.

---

## 3. Problema — DNS não resolvendo no Proxmox

### Sintoma
```bash
curl -fsSL https://tailscale.com/install.sh | sh && tailscale up
# Could not resolve host tailscale.com
```

### Diagnóstico
```bash
ping 8.8.8.8
# Resposta: OK — internet funcionando
# Conclusão: problema de DNS, não de conectividade
```

### Solução — Configurar DNS permanente via Proxmox

```bash
pvesh set /nodes/proxmox/dns --dns1 8.8.8.8 --dns2 8.8.4.4 --search local
```

> **Nota:** O parâmetro `--search` é obrigatório neste comando. Sem ele, retorna erro `400 parameter verification failed`.

**Verificação:**
```bash
ping tailscale.com
# Resposta: OK
```

---

## 4. Problema — Repositórios Enterprise do Proxmox (401 Unauthorized)

### Sintoma
Ao tentar instalar o Tailscale, o `apt update` retornou erros:

```
Err: https://enterprise.proxmox.com/debian/ceph-squid trixie InRelease
401 Unauthorized

Err: https://enterprise.proxmox.com/debian/pve trixie InRelease
401 Unauthorized
```

### Causa
O Proxmox vem configurado por padrão com repositórios **Enterprise**, que exigem licença paga. Para uso pessoal/homelab, é necessário usar os repositórios gratuitos.

### Diagnóstico dos arquivos de repositório

```bash
ls /etc/apt/sources.list.d/
# ceph.list  ceph.sources  debian.sources  pve-enterprise.list  pve-enterprise.sources  tailscale.list
```

Havia **dois formatos** de arquivo de repositório:
- Arquivos `.list` (formato tradicional)
- Arquivos `.sources` (formato moderno — o que estava causando o problema)

### Solução

**Passo 1 — Comentar os arquivos `.list` enterprise:**
```bash
echo "# deb https://enterprise.proxmox.com/debian/pve trixie pve-enterprise" > /etc/apt/sources.list.d/pve-enterprise.list
echo "# deb https://enterprise.proxmox.com/debian/ceph-squid trixie enterprise" > /etc/apt/sources.list.d/ceph.list
```

**Passo 2 — Adicionar repositório gratuito no sources.list principal:**
```bash
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" >> /etc/apt/sources.list
```

> **Atenção:** Verificar se a linha não estava comentada (com `#` na frente). Se estiver, remover o `#`:
> ```bash
> sed -i 's/^# deb https:\/\/download.proxmox.com/deb https:\/\/download.proxmox.com/' /etc/apt/sources.list
> ```

**Passo 3 — Desabilitar os arquivos `.sources` enterprise (causa raiz):**
```bash
echo "Enabled: no" >> /etc/apt/sources.list.d/pve-enterprise.sources
echo "Enabled: no" >> /etc/apt/sources.list.d/ceph.sources
```

> **Observação importante:** Os arquivos `.sources` usam um formato diferente dos `.list`. Não basta comentar as linhas — é necessário adicionar explicitamente `Enabled: no`. Esses arquivos foram a causa raiz do problema persistente.

**Verificação:**
```bash
apt update
# Sem erros 401
```

---

## 5. Instalação do Tailscale e Conexão

```bash
curl -fsSL https://tailscale.com/install.sh | sh && tailscale up
```

- O comando instalou o Tailscale e gerou um link de autenticação
- Link aberto no computador principal, login realizado via Google/GitHub
- Ambos os dispositivos conectados à mesma rede Tailscale

**Verificação de conectividade:**
```bash
# No Proxmox
tailscale ip
# Retorna o IP Tailscale do nó

# No computador principal
ping <IP-TAILSCALE-PROXMOX>
# Resposta: OK

# No Proxmox
ping <IP-TAILSCALE-COMPUTADOR>
# Resposta: OK
```

---

## 6. Acesso à Interface Web do Proxmox

Com o Tailscale funcionando, acessar no navegador do computador principal:

```
https://<IP-TAILSCALE-PROXMOX>:8006
```

> **Lembrete:** O navegador vai exibir um aviso de certificado inválido (autoassinado). Clicar em "Avançado" → "Continuar mesmo assim". No Chrome/Edge, se a opção não aparecer, clicar em qualquer lugar da página de erro e digitar `thisisunsafe`.

---

## 7. Resumo dos Comandos Finais (ordem correta)

```bash
# 1. Configurar DNS permanente
pvesh set /nodes/proxmox/dns --dns1 8.8.8.8 --dns2 8.8.4.4 --search local

# 2. Desabilitar repositórios enterprise (.list)
echo "# deb https://enterprise.proxmox.com/debian/pve trixie pve-enterprise" > /etc/apt/sources.list.d/pve-enterprise.list
echo "# deb https://enterprise.proxmox.com/debian/ceph-squid trixie enterprise" > /etc/apt/sources.list.d/ceph.list

# 3. Desabilitar repositórios enterprise (.sources)
echo "Enabled: no" >> /etc/apt/sources.list.d/pve-enterprise.sources
echo "Enabled: no" >> /etc/apt/sources.list.d/ceph.sources

# 4. Adicionar repositório gratuito
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" >> /etc/apt/sources.list

# 5. Atualizar pacotes
apt update

# 6. Instalar Tailscale
curl -fsSL https://tailscale.com/install.sh | sh && tailscale up
```

---

## 8. Próximos Passos

- [ ] Criar o LXC "apps" no Proxmox
- [ ] Instalar Docker dentro do LXC
- [ ] Configurar Nginx como reverse proxy
- [ ] Criar o primeiro app (finanças ou kanban)
- [ ] Comprar um cabo ethernet para conexão cabeada definitiva do notebook

---

## Lições Aprendidas

1. **Proxmox vem com repositórios enterprise por padrão** — sempre desabilitar em instalações para homelab/uso pessoal. Atenção especial aos arquivos `.sources`, que usam formato diferente dos `.list`.
2. **AP Isolation é comum em roteadores domésticos** — impede comunicação entre WiFi e cabo. Tailscale é uma solução elegante para contornar isso definitivamente.
3. **DNS e conectividade são coisas separadas** — pingar um IP (8.8.8.8) funciona mesmo sem DNS configurado. Sempre testar os dois separadamente.
4. **`pvesh` é a ferramenta correta para configurar o nó Proxmox** — evita edição manual de arquivos que podem ser sobrescritos.
