# Configuração de Wi-Fi no Proxmox VE 9.1
**Hardware:** Dell Inspiron 13 P57G001  
**Adaptador Wi-Fi:** Atheros (driver `ath9k`)  
**OS:** Proxmox VE 9.1 (baseado em Debian 13 Trixie, x86_64)

---

## 1. Diagnóstico inicial

### Listando interfaces de rede
```bash
ip link show
```
Resultado: três interfaces encontradas — `lo`, `wlp1s0`, `vmbr0`.

### Verificando interfaces wireless ativas
```bash
iw dev
```
Resultado: nenhuma saída — a interface Wi-Fi estava desligada.

### Verificando estado da interface Wi-Fi
```bash
ip link show wlp1s0
```
Resultado: estado `DOWN`.

---

## 2. Tentativa de subir a interface

```bash
ip link set wlp1s0 up
ip link show wlp1s0
```
A interface mostrou flags `UP` mas com `NO-CARRIER` e estado `DOWN` — administrativamente ligada, porém sem sinal de rede ainda (esperado nesse ponto).

---

## 3. Investigação de bloqueios

### Verificando RF Kill (bloqueio de rádio)
```bash
cat /sys/class/rfkill/rfkill0/soft   # retornou 0 (desbloqueado)
cat /sys/class/rfkill/rfkill0/hard   # retornou 0 (desbloqueado)
```
Nenhum bloqueio de hardware ou software.

### Verificando logs do kernel
```bash
dmesg | grep -i ath9k
```
Resultado relevante:
```
ath9k 0000:01:00.0 wlp1s0: renamed from wlan0
```
O driver `ath9k` carregou corretamente. O kernel apenas renomeou a interface de `wlan0` para `wlp1s0` (padrão *predictable network interface names*).

```bash
dmesg | tail -50 | grep -iE "wlp1s0|ath9k|error|fail|warn"
```
Erros encontrados eram do `ath3k` (Bluetooth), não do Wi-Fi:
```
usb 1-1.6: direct firmware load for ar3k/AthrBT_0x31010000.dfu failed with error -2
Bluetooth: Loading patch file failed
ath3k 1-1.6:1.0: probe with driver ath3k failed with error -2
```
O Wi-Fi estava limpo.

---

## 4. Instalação do wpa_supplicant via pendrive

### Problema
O Proxmox não tinha repositórios configurados em `/etc/apt/sources.list` e não havia conexão com internet (sem cabo disponível, USB tethering indisponível).

### Solução
Download manual dos pacotes `.deb` em outro computador (Windows) e transferência via pendrive formatado em FAT32.

### Pacotes baixados (Debian Trixie amd64)
| Pacote | URL |
|---|---|
| `wpasupplicant_2.10-24_amd64.deb` | `http://ftp.br.debian.org/debian/pool/main/w/wpa/` |
| `libnl-genl-3-200_3.7.0-2_amd64.deb` | `http://ftp.br.debian.org/debian/pool/main/libn/libnl3/` |
| `libpcsclite1_2.3.3-1_amd64.deb` | `http://ftp.br.debian.org/debian/pool/main/p/pcsc-lite/` |

### Montagem do pendrive
```bash
mkdir /mnt/usb
mount /dev/sdc1 /mnt/usb
ls /mnt/usb
```

### Instalação (ordem importa — dependências primeiro)
```bash
dpkg -i /mnt/usb/libnl-genl-3-200_3.7.0-2_amd64.deb
dpkg -i /mnt/usb/libpcsclite1_2.3.3-1_amd64.deb
dpkg -i /mnt/usb/wpasupplicant_2.10-24_amd64.deb
```

---

## 5. Configuração do wpa_supplicant

### Criando o arquivo de configuração
```bash
wpa_passphrase "NOME_DA_REDE" "SENHA" > /etc/wpa_supplicant/wpa_supplicant.conf
```

### Adicionando interface de controle (necessária para o wpa_cli funcionar)
Editado `/etc/wpa_supplicant/wpa_supplicant.conf` para incluir no início:
```
ctrl_interface=/run/wpa_supplicant
ctrl_interface_group=0
```

### Iniciando o wpa_supplicant em background
```bash
wpa_supplicant -B -i wlp1s0 -c /etc/wpa_supplicant/wpa_supplicant.conf
```

### Verificando autenticação
```bash
wpa_cli -i wlp1s0 status
```
Resultado: `wpa_state=COMPLETED` — autenticado com sucesso.

---

## 6. Obtendo endereço IP e corrigindo rota

### Obtendo IP via DHCP
```bash
dhclient wlp1s0
```
IP obtido: `192.168.15.25`

### Problema de rota
```bash
ip route show
```
Resultado:
```
default via 192.168.15.1 dev vmbr0 proto kernel onlink
192.168.15.0/24 dev vmbr0 proto kernel scope link src 192.168.15.2
192.168.15.0/24 dev wlp1s0 proto kernel scope link src 192.168.15.25
```
A rota padrão estava apontando para `vmbr0` (bridge sem internet) em vez de `wlp1s0`.

### Corrigindo a rota manualmente
```bash
ip route del default
ip route add default via 192.168.15.1 dev wlp1s0
```

### Testando conectividade
```bash
ping -c 4 8.8.8.8
```
Sucesso.

---

## 7. Tornando a configuração persistente

Editado `/etc/network/interfaces`:

```
auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.15.2/24
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0

auto wlp1s0
iface wlp1s0 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

source /etc/network/interfaces.d/*
```

> **Nota:** O `gateway 192.168.15.1` foi removido do bloco `vmbr0` pois a bridge não tem acesso à internet — isso causava o conflito de rota padrão.

### Reboot e validação
```bash
reboot
# após reiniciar:
ping -c 4 8.8.8.8
```
Wi-Fi conectando automaticamente após reboot. ✓
