# Guia completa: Configuració de xarxa amb Incus
## Router Alpine + Servidor DNS/DHCP Ubuntu + Client Ubuntu

---

## 🎯 Introducció i objectius

En aquesta pràctica aprendràs a crear una xarxa virtualitzada amb contenidors niats utilitzant Incus a Google Cloud Platform. Crearem:

- **Router Alpine**: Farà de porta d'enllaç entre la xarxa interna i l'exterior
- **Servidor Ubuntu**: Proporcionarà serveis DHCP i DNS
- **Client Ubuntu**: Consumidor dels serveis, configurarà la xarxa automàticament via DHCP

**Conceptes que aprendràs:**
- Creació i configuració de compte Google Cloud
- Creació de VMs amb virtualització niada
- Contenidors niats (nested containers)
- Configuració de xarxes en Incus
- Enrutament i NAT en Linux
- Serveis DHCP i DNS amb dnsmasq

---

## ☁️ PART 1: Configuració inicial de Google Cloud Platform

### Pas 1: Crear un compte de Google

Si ja tens un compte de Gmail, pots saltar al Pas 2.

1. Visita https://accounts.google.com
2. Fes clic a **"Crear cuenta"**
3. Omple les dades:
   - Nom i cognom
   - Nom d'usuari (serà el teu @gmail.com)
   - Contrasenya segura
4. Verifica el teu telèfon
5. Accepta els termes i condicions

### Pas 2: Activar Google Cloud Platform

1. Visita https://console.cloud.google.com
2. Fes clic a **"Try for free"** o **"Probar gratis"**
3. Inicia sessió amb el teu compte de Google
4. Selecciona el teu país: **España**
5. Accepta els termes de servei

**📝 Important:** Google Cloud ofereix 300$ de crèdit gratuït per a nous usuaris durant 90 dies.

### Pas 3: Configurar la facturació

1. Se't demanarà informació de pagament:
   - **Tipus de compte**: Particular o Empresa
   - **Dades de contacte**
   - **Targeta de crèdit/dèbit** (no se't cobrarà durant el període de prova)
2. Fes clic a **"Iniciar mi prueba gratuita"**

**⚠️ Tranquil:** Durant el període de prova, Google no cobrarà cap quantitat sense el teu consentiment explícit.

### Pas 4: Crear un nou projecte

1. A la consola de Google Cloud, fes clic al selector de projectes (a dalt a l'esquerra, al costat del logo de Google Cloud)
2. Fes clic a **"NUEVO PROYECTO"** o **"NEW PROJECT"**
3. Omple les dades:
   - **Nom del projecte**: `Practica-Incus-Xarxes` (o el que preferisques)
   - **Organización**: Deixa el valor per defecte
4. Fes clic a **"CREAR"**
5. Espera uns segons fins que es cree el projecte
6. Assegura't que el nou projecte està seleccionat al selector de projectes

---

## 🖥️ PART 2: Creació de la màquina virtual amb virtualització niada

### Pas 5: Activar l'API de Compute Engine

1. Al menú lateral esquerre (☰), fes clic a **"Compute Engine"**
2. Fes clic a **"Instancias de VM"** o **"VM instances"**
3. Si és la primera vegada, hauràs d'esperar que s'active l'API de Compute Engine
4. Fes clic a **"HABILITAR"** o **"ENABLE"** i espera 1-2 minuts

### Pas 6: Obrir Cloud Shell (Terminal al navegador)

**⚠️ IMPORTANT:** La interfície gràfica de Google Cloud NO permet activar la virtualització niada, per tant **hem d'usar Cloud Shell (terminal)**.

1. A la part superior dreta de la consola, localitza la icona de terminal: **`>_`** (al costat de la icona de cerca i la campana)
2. Fes clic a la icona **`>_`** per obrir Cloud Shell
3. S'obrirà un terminal a la part inferior de la pantalla
4. Espera que es carregue (primera vegada pot trigar 30 segons)
5. Veuràs un prompt com: `nomdusuari@cloudshell:~ (lxd-test-476020)$`

**📝 Explicació:** Cloud Shell és un terminal Linux gratuït que Google proporciona per gestionar recursos. Ja té preinstal·lat `gcloud` i està autenticat automàticament.

### Pas 7: Obtenir el teu Project ID

Abans de crear la VM, necessitem el teu Project ID. Al terminal de Cloud Shell, executa:

```bash
# Veure el Project ID actual
gcloud config get-value project
```

**Apunta aquest ID**, l'usaràs a la següent comanda. Exemple: `lxd-test-476020`

### Pas 8: Crear la VM amb virtualització niada

**Ara ve la part important!** Copia i enganxa la següent comanda al Cloud Shell. **ABANS d'executar-la, canvia:**

- `lxd-test-476020` → El teu Project ID (apareix 1 vegada a la comanda)
- `ubuntulxd` → Pots deixar aquest nom o canviar-lo

```bash
gcloud compute instances create ubuntulxd \
  --project=lxd-test-476020 \
  --enable-nested-virtualization \
  --zone=europe-southwest1-c \
  --machine-type=n2-standard-4 \
  --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default \
  --maintenance-policy=MIGRATE \
  --provisioning-model=STANDARD \
  --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \
  --create-disk=auto-delete=yes,boot=yes,device-name=ubuntulxd,image=projects/ubuntu-os-cloud/global/images/ubuntu-2404-noble-amd64-v20241004,mode=rw,size=200,type=pd-balanced \
  --no-shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring
```

**📝 Explicació dels paràmetres importants:**
- `--enable-nested-virtualization`: **CLAU!** Activa la virtualització niada
- `--zone=europe-southwest1-c`: Zona de Madrid (la més propera)
- `--machine-type=n2-standard-4`: 4 vCPUs + 16 GB RAM
- `--create-disk=...size=200`: Disc de 200 GB
- `image=ubuntu-2404-noble`: Ubuntu 24.04 LTS

### Pas 9: Esperar que es cree la VM

Després d'executar la comanda, veuràs:

```
Created [https://www.googleapis.com/compute/v1/projects/lxd-test-476020/zones/europe-southwest1-c/instances/ubuntulxd].
NAME       ZONE                   MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
ubuntulxd  europe-southwest1-c    n2-standard-4               10.204.0.2   34.175.23.45   RUNNING
```

**🎉 Felicitats!** La teua VM està creada i en execució amb virtualització niada activada.

### Pas 10: Verificar la creació a la interfície gràfica

1. Torna a la pestanya **"Instancias de VM"** del navegador
2. Fes clic al botó **"ACTUALIZAR"** o refresca la pàgina
3. Hauríes de veure la teua VM `ubuntulxd` amb:
   - **Estado**: Marca verda ✓ (Running)
   - **Zona**: `europe-southwest1-c`
   - **Tipo de máquina**: `n2-standard-4`
   - **IP externa**: Una IP pública (ex: 34.175.x.x)
   - **IP interna**: Una IP privada (ex: 10.204.x.x)

---

## 🔌 PART 3: Connectar-se a la màquina virtual

### Pas 11: Tancar Cloud Shell (opcional)

Pots tancar el Cloud Shell fent clic a la **X** de la finestra del terminal (part inferior) o deixar-lo obert per a futures comandes.

### Pas 12: Connectar-se per SSH des de la interfície web

1. A la llista d'**"Instancias de VM"**, localitza `ubuntulxd`
2. A la columna **"Conectar"**, fes clic al botó **"SSH"** (botó desplegable)
3. Selecciona **"Abrir en una ventana del navegador"**
4. S'obrirà una nova finestra/pestanya amb una consola SSH
5. Espera que es carregue i establisca la connexió (primera vegada pot trigar 30-60 segons)
6. Pot aparèixer un missatge sobre claus SSH, escriu **yes** i prem Enter

Veuràs un terminal amb algo com:

```
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-1018-gcp x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

nomdusuari@ubuntulxd:~$
```

**🎉 Felicitats! Ja estàs dins de la teua màquina virtual!**

### Pas 13: Verificar la virtualització niada

**Aquest és un pas CRÍTIC.** Comprova que la virtualització niada està activada correctament:

```bash
# Comprovar que KVM està disponible
ls -la /dev/kvm
```

**Sortida esperada:**
```
crw-rw---- 1 root kvm 10, 232 Oct 26 10:00 /dev/kvm
```

✅ **Si veus aquest fitxer:** La virtualització niada funciona! Pots continuar.

❌ **Si veus "No such file or directory":** Algo ha anat malament. Possibles solucions:

1. Espera 2-3 minuts i torna a comprovar (a vegades triga a carregar)
2. Reinicia la VM des de la consola de Google Cloud
3. Si encara no funciona, elimina la VM i torna a crear-la amb la comanda del Pas 8, assegurant-te que incloem `--enable-nested-virtualization`

### Pas 14: Comprovar els recursos de la VM

```bash
# Veure la quantitat de CPU i RAM
lscpu | grep -E "^CPU\(s\)|Model name"
free -h

# Veure l'espai en disc
df -h /
```

**Sortida esperada:**
- **CPU**: 4 cores (n2-standard-4)
- **RAM**: ~15GB disponible (16GB total, el sistema usa ~1GB)
- **Disc**: ~196GB disponible (de 200GB)

---

## 🖥️ PART 4: Instal·lació d'Incus al host

### Pas 15: Instal·lar Incus

**Estàs al terminal SSH de la teua VM `ubuntulxd`.**

```bash
# Afegir el repositori d'Incus
curl -fsSL https://pkgs.zabbly.com/key.asc | sudo gpg --dearmor -o /usr/share/keyrings/zabbly.gpg

echo "deb [signed-by=/usr/share/keyrings/zabbly.gpg] https://pkgs.zabbly.com/incus/stable $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/zabbly-incus-stable.list

# Actualitzar i instal·lar
sudo apt update
sudo apt install -y incus

# Afegir el teu usuari al grup incus-admin
sudo usermod -aG incus-admin $USER
```

### Pas 16: Reiniciar la sessió SSH

**⚠️ Important:** Has de tancar i tornar a obrir la connexió SSH perquè el grup s'apliqui.

1. Tanca la finestra SSH del navegador (o escriu `exit`)
2. Torna a la consola de Google Cloud
3. Fes clic de nou al botó **"SSH"** al costat de `ubuntulxd`
4. Espera que es reconnecte

### Pas 17: Verificar que el grup està aplicat

```bash
# Comprovar que pertanys al grup incus-admin
groups

# Hauries de veure "incus-admin" a la llista
```

---

## ⚙️ PART 5: Configuració d'Incus

### Pas 18: Inicialitzar Incus amb Btrfs

```bash
sudo incus admin init
```

Respon així a les preguntes (prem **Enter** després de cada resposta):

```
Would you like to use clustering? (yes/no) [default=no]: 
↪ no

Do you want to configure a new storage pool? (yes/no) [default=yes]: 
↪ yes

Name of the storage pool [default=default]: 
↪ (prem Enter per defecte)

Name of the storage backend (btrfs, dir, lvm, zfs) [default=zfs]: 
↪ btrfs

Create a new BTRFS pool? (yes/no) [default=yes]: 
↪ yes

Would you like to use an existing empty block device? (yes/no) [default=no]: 
↪ no

Size in GiB of the new loop device (1GiB minimum) [default=30GiB]: 
↪ 150GiB

Would you like to connect to a MAAS server? (yes/no) [default=no]: 
↪ no

Would you like to create a new local network bridge? (yes/no) [default=yes]: 
↪ yes

What should the new bridge be called? [default=incusbr0]: 
↪ (prem Enter per defecte)

What IPv4 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]: 
↪ auto

What IPv6 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]: 
↪ none

Would you like the server to be available over the network? (yes/no) [default=no]: 
↪ no

Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: 
↪ yes

Would you like a YAML "incus admin init" preseed to be printed? (yes/no) [default=no]:
↪ no
```

**📝 Explicació:**
- **Btrfs**: Sistema de fitxers modern amb snapshots
- **150GiB**: Espai per als contenidors (tens 200GB, deixem 50GB per al sistema)
- **incusbr0**: Bridge que connecta contenidors amb internet
- **auto**: Incus crearà una xarxa privada automàticament (10.x.x.x/24)

### Pas 19: Verificar la instal·lació

```bash
# Llistar contenidors (hauria d'estar buit)
incus list

# Llistar xarxes
incus network list

# Hauries de veure incusbr0 amb una IP com 10.162.70.1/24
```

---

## 📦 PART 6: Creació del contenidor pare

### Pas 20: Crear el contenidor pare amb nesting activat

```bash
incus launch images:ubuntu/24.04 contenidor-pare \
  -c security.nesting=true \
  -c security.privileged=true
```

**📝 Explicació:**
- `security.nesting=true`: Permet contenidors dins d'aquest contenidor
- `security.privileged=true`: Dona permisos necessaris per al nesting

### Pas 21: Esperar i verificar

```bash
# Esperar 10 segons que arranque
sleep 10

# Veure l'estat
incus list
```

Hauries de veure:

```
+------------------+---------+----------------------+------+-----------+
|       NAME       |  STATE  |         IPV4         | IPV6 |   TYPE    |
+------------------+---------+----------------------+------+-----------+
| contenidor-pare  | RUNNING | 10.162.70.23 (eth0)  |      | CONTAINER |
+------------------+---------+----------------------+------+-----------+
```

### Pas 22: Instal·lar Incus dins del contenidor pare

```bash
# Accedir al contenidor
incus exec contenidor-pare -- bash

# Dins del contenidor pare, instal·lar Incus
apt update
apt install -y incus

# Inicialitzar Incus (prem Enter a tot per defecte)
incus admin init
```

Quan et demane opcions, prem **Enter** a totes les preguntes per acceptar els valors per defecte.

**✅ Ara tens Incus dins d'Incus!**

---

## 🌐 PART 7: Configuració de la topologia de xarxa

**Estàs dins del contenidor pare.** Ara crearem dues xarxes:

1. **incusbr-externa**: Connecta el router amb l'exterior
2. **incusbr-interna**: Xarxa privada entre router, servidor i client

### Pas 23: Crear les xarxes

```bash
# Xarxa externa (amb NAT per eixir a internet)
incus network create incusbr-externa \
  ipv4.address=10.200.0.1/24 \
  ipv4.nat=true \
  ipv6.address=none

# Xarxa interna (aïllada, sense DHCP)
incus network create incusbr-interna \
  ipv4.address=none \
  ipv4.nat=false \
  ipv6.address=none \
  ipv4.dhcp=false \
  ipv6.dhcp=false
```

**📝 Explicació:**
- **incusbr-externa**: Té NAT, els contenidors podran accedir a internet
- **incusbr-interna**: Xarxa "muda", la configurarem manualment

### Pas 24: Verificar les xarxes

```bash
incus network list
```

Hauries de veure:

```
+------------------+----------+---------+-----------------+
|       NAME       |   TYPE   | MANAGED |      IPV4       |
+------------------+----------+---------+-----------------+
| incusbr-externa  | bridge   | YES     | 10.200.0.1/24   |
| incusbr-interna  | bridge   | YES     | none            |
+------------------+----------+---------+-----------------+
```

---

## 🔀 PART 8: Configuració del Router Alpine

### Pas 25: Crear el contenidor Alpine amb dues interfícies

```bash
# Crear el contenidor Alpine
incus launch images:alpine/3.19 router-alpine

# Afegir la segona interfície (interna)
incus config device add router-alpine eth1 nic \
  nictype=bridged \
  parent=incusbr-interna \
  name=eth1

# Reiniciar per aplicar
incus restart router-alpine
sleep 5
```

**📝 Explicació:**
- Alpine es connecta per defecte a `incusbr-externa` (eth0)
- Afegim `eth1` connectada a `incusbr-interna`

### Pas 26: Configurar el router Alpine

```bash
# Accedir al router
incus exec router-alpine -- sh
```

**Ara estàs dins del router Alpine.** Executa:

```bash
# Instal·lar eines necessàries
apk add iptables ip6tables

# Configurar la interfície interna amb IP estàtica
cat > /etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 192.168.100.1
    netmask 255.255.255.0
EOF

# Reiniciar la xarxa
rc-service networking restart

# Activar IP forwarding (enrutament)
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# Configurar NAT per a la xarxa interna
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE

# Guardar les regles d'iptables
apk add iptables-openrc
/etc/init.d/iptables save
rc-update add iptables
```

**📝 Explicació:**
- **192.168.100.1/24**: IP del router a la xarxa interna
- **IP forwarding**: Permet passar paquets entre xarxes
- **MASQUERADE**: NAT que tradueix IPs privades a públiques

### Pas 27: Verificar configuració del router

```bash
# Veure les interfícies
ip addr show

# Hauries de veure:
# eth0: 10.200.0.X/24 (DHCP de incusbr-externa)
# eth1: 192.168.100.1/24 (estàtica)

# Verificar IP forwarding
cat /proc/sys/net/ipv4/ip_forward
# Ha de mostrar: 1

# Provar internet
ping -c 3 8.8.8.8

# Sortir del router
exit
```

---

## 🖧 PART 9: Configuració del Servidor DNS/DHCP Ubuntu

### Pas 28: Crear el contenidor servidor

```bash
# Crear el servidor a la xarxa interna
incus launch images:ubuntu/24.04 servidor-ubuntu --network incusbr-interna

# Accedir al servidor
incus exec servidor-ubuntu -- bash
```

### Pas 29: Configurar IP estàtica

**Estàs dins del servidor Ubuntu.**

```bash
# Instal·lar eines
apt update
apt install -y netplan.io dnsmasq iputils-ping

# Desactivar dnsmasq per configurar-lo
systemctl stop dnsmasq
systemctl disable dnsmasq

# Configurar Netplan amb IP estàtica
cat > /etc/netplan/10-incus.yaml << 'EOF'
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.100.10/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
EOF

# Aplicar configuració
netplan apply
```

**📝 Explicació:**
- **192.168.100.10**: IP estàtica del servidor
- **via 192.168.100.1**: El router és la porta d'enllaç
- **8.8.8.8**: DNS de Google (temporal)

### Pas 30: Configurar dnsmasq (DHCP + DNS)

```bash
# Backup de la configuració original
mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak

# Nova configuració
cat > /etc/dnsmasq.conf << 'EOF'
# No llegir /etc/resolv.conf
no-resolv

# Servidors DNS upstreams
server=8.8.8.8
server=8.8.4.4

# Interfície on escoltar
interface=eth0
bind-interfaces

# Rang DHCP: de .50 a .200, lloguer de 12 hores
dhcp-range=192.168.100.50,192.168.100.200,12h

# Opcions DHCP
dhcp-option=3,192.168.100.1        # Gateway
dhcp-option=6,192.168.100.10       # DNS server

# Domini local
domain=xarxa.local
local=/xarxa.local/

# Registres DNS estàtics
address=/router.xarxa.local/192.168.100.1
address=/servidor.xarxa.local/192.168.100.10

# Logs
log-queries
log-dhcp
EOF

# Activar i iniciar
systemctl enable dnsmasq
systemctl start dnsmasq
```

**📝 Explicació:**
- **dhcp-range**: IPs assignables als clients (.50 - .200)
- **dhcp-option=3**: Gateway (router)
- **dhcp-option=6**: Servidor DNS (nosaltres)
- **xarxa.local**: Domini local

### Pas 31: Verificar el servei

```bash
# Veure estat
systemctl status dnsmasq

# Provar internet
ping -c 3 google.com

# Sortir del servidor
exit
```

---

## 💻 PART 10: Configuració del Client Ubuntu

### Pas 32: Crear el contenidor client

```bash
# Crear el client a la xarxa interna
incus launch images:ubuntu/24.04 client-ubuntu --network incusbr-interna

# Accedir al client
incus exec client-ubuntu -- bash
```

### Pas 33: Configurar DHCP client

**Estàs dins del client Ubuntu.**

```bash
# Instal·lar eines
apt update
apt install -y netplan.io iputils-ping dnsutils curl

# Configurar Netplan per DHCP
cat > /etc/netplan/10-incus.yaml << 'EOF'
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
EOF

# Aplicar
netplan apply

# Esperar DHCP
sleep 5
```

### Pas 34: Verificar configuració rebuda

```bash
# Veure IP assignada
ip addr show eth0

# Hauries de veure: 192.168.100.X/24 (entre .50 i .200)

# Veure ruta per defecte
ip route show

# Hauria de mostrar: default via 192.168.100.1

# Veure DNS
cat /etc/resolv.conf

# Hauria de mostrar: nameserver 192.168.100.10
```

---

## ✅ PART 11: Proves de connectivitat

### Pas 35: Provar tot el stack de xarxa

**Encara estàs dins del client Ubuntu.**

```bash
echo "=== Prova 1: Ping al router ==="
ping -c 3 192.168.100.1

echo ""
echo "=== Prova 2: Ping al servidor DNS/DHCP ==="
ping -c 3 192.168.100.10

echo ""
echo "=== Prova 3: Resolució DNS local ==="
nslookup router.xarxa.local
nslookup servidor.xarxa.local

echo ""
echo "=== Prova 4: Resolució DNS externa ==="
nslookup google.com

echo ""
echo "=== Prova 5: Ping a internet ==="
ping -c 3 8.8.8.8

echo ""
echo "=== Prova 6: Accés HTTP a internet ==="
curl -I https://www.google.com
```

**Si tot funciona:**
- ✅ Tots els pings responen
- ✅ Les resolucions DNS funcionen
- ✅ Pots accedir a internet

### Pas 36: Veure logs del servidor DHCP

```bash
# Sortir del client
exit

# Accedir al servidor
incus exec servidor-ubuntu -- bash

# Veure logs
tail -n 50 /var/log/syslog | grep dnsmasq

# Hauries de veure les peticions DHCP i DNS del client
```

**Exemple de logs:**

```
dnsmasq-dhcp[123]: DHCPDISCOVER(eth0) aa:bb:cc:dd:ee:ff
dnsmasq-dhcp[123]: DHCPOFFER(eth0) 192.168.100.50 aa:bb:cc:dd:ee:ff
dnsmasq-dhcp[123]: DHCPREQUEST(eth0) 192.168.100.50 aa:bb:cc:dd:ee:ff
dnsmasq-dhcp[123]: DHCPACK(eth0) 192.168.100.50 aa:bb:cc:dd:ee:ff client-ubuntu
dnsmasq[123]: query[A] google.com from 192.168.100.50
```

```bash
# Sortir del servidor
exit
```

---

## 📊 Esquemes de l'estructura final

### Esquema 1: Jerarquia de contenidors

```
┌─────────────────────────────────────────────────────────────┐
│  Google Cloud Platform - Zona Madrid (europe-southwest1-c)  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  VM: ubuntulxd (Ubuntu 24.04)                         │  │
│  │  • 4 vCPU (N2)                                        │  │
│  │  • 16 GB RAM                                          │  │
│  │  • 200 GB Disc                                        │  │
│  │  • Virtualització niada activada                      │  │
│  │  • IP externa: 34.175.x.x                             │  │
│  │  • IP interna: 10.204.x.x                             │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────────────┐    │  │
│  │  │  Incus (Host Level 1)                        │    │  │
│  │  │  • Storage pool: Btrfs 150GB                 │    │  │
│  │  │  • Network: incusbr0 (10.162.70.0/24)        │    │  │
│  │  │                                               │    │  │
│  │  │  ┌────────────────────────────────────────┐  │    │  │
│  │  │  │  Contenidor: contenidor-pare           │  │    │  │
│  │  │  │  (Ubuntu 24.04)                        │  │    │  │
│  │  │  │  • security.nesting=true               │  │    │  │
│  │  │  │  • security.privileged=true            │  │    │  │
│  │  │  │  • IP: 10.162.70.23                    │  │    │  │
│  │  │  │                                         │  │    │  │
│  │  │  │  ┌───────────────────────────────────┐ │  │    │  │
│  │  │  │  │  Incus (Nested Level 2)           │ │  │    │  │
│  │  │  │  │                                    │ │  │    │  │
│  │  │  │  │  Networks:                         │ │  │    │  │
│  │  │  │  │  • incusbr-externa (10.200.0.0/24) │ │  │    │  │
│  │  │  │  │  • incusbr-interna (192.168.100.0) │ │  │    │  │
│  │  │  │  │                                    │ │  │    │  │
│  │  │  │  │  Contenidors:                      │ │  │    │  │
│  │  │  │  │  ┌────────────────────────────┐   │ │  │    │  │
│  │  │  │  │  │  router-alpine             │   │ │  │    │  │
│  │  │  │  │  │  eth0: 10.200.0.X          │   │ │  │    │  │
│  │  │  │  │  │  eth1: 192.168.100.1       │   │ │  │    │  │
│  │  │  │  │  │  (Router + NAT)            │   │ │  │    │  │
│  │  │  │  │  └────────────────────────────┘   │ │  │    │  │
│  │  │  │  │  ┌────────────────────────────┐   │ │  │    │  │
│  │  │  │  │  │  servidor-ubuntu           │   │ │  │    │  │
│  │  │  │  │  │  eth0: 192.168.100.10      │   │ │  │    │  │
│  │  │  │  │  │  (DHCP + DNS)              │   │ │  │    │  │
│  │  │  │  │  └────────────────────────────┘   │ │  │    │  │
│  │  │  │  │  ┌────────────────────────────┐   │ │  │    │  │
│  │  │  │  │  │  client-ubuntu             │   │ │  │    │  │
│  │  │  │  │  │  eth0: 192.168.100.50-200  │   │ │  │    │  │
│  │  │  │  │  │  (DHCP Client)             │   │ │  │    │  │
│  │  │  │  │  └────────────────────────────┘   │ │  │    │  │
│  │  │  │  └───────────────────────────────────┘ │  │    │  │
│  │  │  └────────────────────────────────────────┘  │    │  │
│  │  └──────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Esquema 2: Flux de xarxa i connectivitat

```
                    INTERNET
                       ↕
         ┌─────────────────────────┐
         │   Google Cloud VPC      │
         │   (Xarxa de GCP)        │
         └─────────────────────────┘
                       ↕
         ┌─────────────────────────┐
         │  VM: ubuntulxd          │
         │  IP externa: 34.175.x.x │
         │  IP interna: 10.204.x.x │
         └─────────────────────────┘
                       ↕
         ┌─────────────────────────┐
         │  Incus Host (Level 1)   │
         │  incusbr0: 10.162.70.1  │
         └─────────────────────────┘
                       ↕
         ┌─────────────────────────┐
         │  contenidor-pare        │
         │  IP: 10.162.70.23       │
         └─────────────────────────┘
                       ↕
         ┌─────────────────────────┐
         │  Incus Nested (Level 2) │
         └─────────────────────────┘
                   ↙       ↘
    incusbr-externa      incusbr-interna
    (10.200.0.0/24)      (192.168.100.0/24)
           ↓                    ↓
    ┌──────────────┐     ┌─────────────────────────┐
    │ router-alpine│←────→│                         │
    │ eth0: DHCP   │     │ eth1: 192.168.100.1     │
    │ (10.200.0.X) │     │ (Gateway + NAT)         │
    └──────────────┘     └─────────────────────────┘
                                   ↓
                         ┌─────────┴──────────┐
                         ↓                    ↓
              ┌──────────────────┐   ┌───────────────────┐
              │ servidor-ubuntu  │   │  client-ubuntu    │
              │ 192.168.100.10   │   │  192.168.100.X    │
              │ (DHCP + DNS)     │   │  (via DHCP)       │
              └──────────────────┘   └───────────────────┘
```

### Esquema 3: Flux de paquets - Client accedint a Internet

```
1. Client vol accedir a google.com
   client-ubuntu (192.168.100.50)
        │
        │ DNS query: google.com?
        ↓
   servidor-ubuntu (192.168.100.10)
        │
        │ Consulta a 8.8.8.8
        │ Resposta: 142.250.x.x
        ↓
   client-ubuntu rep la IP
        │
        │ Paquet destinació: 142.250.x.x
        ↓
   router-alpine (192.168.100.1)
        │
        │ NAT: Tradueix 192.168.100.50 → 10.200.0.X
        ↓
   incusbr-externa (10.200.0.0/24)
        │
        │ NAT: Tradueix 10.200.0.X → 10.162.70.23
        ↓
   contenidor-pare (10.162.70.23)
        │
        │ NAT: Tradueix 10.162.70.23 → 10.204.x.x
        ↓
   VM ubuntulxd (10.204.x.x)
        │
        │ NAT de GCP: Tradueix 10.204.x.x → 34.175.x.x
        ↓
   INTERNET → google.com

   La resposta segueix el camí invers
```

### Esquema 4: Serveis DHCP/DNS

```
┌────────────────────────────────────────────────────────┐
│            servidor-ubuntu (192.168.100.10)            │
│                     dnsmasq                            │
├────────────────────────────────────────────────────────┤
│                                                        │
│  SERVEI DHCP:                                          │
│  • Rang: 192.168.100.50 - 192.168.100.200             │
│  • Lloguer: 12 hores                                   │
│  • Gateway: 192.168.100.1 (router-alpine)             │
│  • DNS: 192.168.100.10 (ell mateix)                    │
│                                                        │
│  SERVEI DNS:                                           │
│  • Domini local: xarxa.local                           │
│  • Registres locals:                                   │
│    - router.xarxa.local → 192.168.100.1               │
│    - servidor.xarxa.local → 192.168.100.10            │
│  • DNS upstream: 8.8.8.8, 8.8.4.4                      │
│                                                        │
└────────────────────────────────────────────────────────┘
           ↓ Assigna IP              ↓ Resol noms
    ┌──────────────┐          ┌──────────────┐
    │ client-ubuntu│          │ client-ubuntu│
    │ DHCP Client  │          │ DNS Client   │
    └──────────────┘          └──────────────┘
```

---

## 🎓 Conceptes apresos

### Virtualització i contenidors
- ✅ Creació de VMs amb virtualització niada a Google Cloud
- ✅ Contenidors niats (contenidors dins de contenidors)
- ✅ Diferència entre VM i contenidor
- ✅ Avantatges de la virtualització niada per a entorns de proves

### Xarxes
- ✅ Creació i configuració de bridges de xarxa
- ✅ Xarxes aïllades vs xarxes amb NAT
- ✅ Assignació d'IPs estàtiques i dinàmiques
- ✅ Configuració de rutes i gateways

### Enrutament
- ✅ IP Forwarding: permetre que un host enrute paquets
- ✅ NAT/Masquerading: traducció d'adreces de xarxa
- ✅ Configuració d'iptables per al NAT
- ✅ Múltiples nivells de NAT

### Serveis de xarxa
- ✅ DHCP: assignació automàtica d'IPs
- ✅ DNS: resolució de noms locals i externs
- ✅ dnsmasq: servidor lleuger DHCP+DNS
- ✅ Configuració de dominis locals

### Eines i sistemes
- ✅ Google Cloud Platform: creació i gestió de VMs
- ✅ Cloud Shell: terminal integrat al navegador
- ✅ Incus: gestor de contenidors modern
- ✅ Alpine Linux: distribució lleugera per a routers
- ✅ Ubuntu Server: serveis d'infraestructura
- ✅ Netplan: configuració de xarxa en Ubuntu

---

## 🔧 Comandes útils per a gestionar la pràctica

### Gestió de la VM a Google Cloud (des de Cloud Shell)

```bash
# Aturar la VM
gcloud compute instances stop ubuntulxd --zone=europe-southwest1-c

# Iniciar la VM
gcloud compute instances start ubuntulxd --zone=europe-southwest1-c

# Veure l'estat
gcloud compute instances list --filter="name=ubuntulxd"

# Eliminar la VM (ESBORRARÀ TOT!)
gcloud compute instances delete ubuntulxd --zone=europe-southwest1-c
```

**💡 Consell:** Pots aturar la VM quan no l'usis per estalviar crèdit.

### Llistar contenidors

```bash
# Al host (VM ubuntulxd)
incus list

# Dins del contenidor pare
incus exec contenidor-pare -- incus list
```

### Reiniciar contenidors

```bash
# Dins del contenidor pare
incus restart router-alpine
incus restart servidor-ubuntu
incus restart client-ubuntu
```

### Veure logs dels serveis

```bash
# Logs del servidor DHCP/DNS
incus exec contenidor-pare -- incus exec servidor-ubuntu -- tail -f /var/log/syslog | grep dnsmasq

# Logs del router (iptables)
incus exec contenidor-pare -- incus exec router-alpine -- dmesg | grep -i iptables
```

### Comprovar connectivitat

```bash
# Des del client, provar tot
incus exec contenidor-pare -- incus exec client-ubuntu -- bash -c "
  echo '=== Ping router ==='
  ping -c 2 192.168.100.1
  echo '=== Ping servidor ==='
  ping -c 2 192.168.100.10
  echo '=== DNS local ==='
  nslookup router.xarxa.local
  echo '=== DNS extern ==='
  nslookup google.com
  echo '=== Internet ==='
  ping -c 2 8.8.8.8
"
```

### Clonar configuracions

```bash
# Dins del contenidor pare
incus copy servidor-ubuntu servidor-backup
incus copy client-ubuntu client-backup
```

### Aturar i esborrar tot (neteja completa)

```bash
# Dins del contenidor pare
incus stop router-alpine servidor-ubuntu client-ubuntu
incus delete router-alpine servidor-ubuntu client-ubuntu
incus network delete incusbr-externa incusbr-interna

# Sortir del contenidor pare
exit

# Al host
incus stop contenidor-pare
incus delete contenidor-pare
```

---

## 🐛 Resolució de problemes comuns

### Problema 1: El client no obté IP per DHCP

**Símptomes:**
```bash
ip addr show eth0
# No mostra cap IP assignada
```

**Solucions:**

1. Verifica que dnsmasq està en execució al servidor:
```bash
incus exec contenidor-pare -- incus exec servidor-ubuntu -- systemctl status dnsmasq
```

2. Revisa els logs de dnsmasq:
```bash
incus exec contenidor-pare -- incus exec servidor-ubuntu -- tail -n 50 /var/log/syslog | grep dnsmasq
```

3. Força la renovació DHCP al client:
```bash
incus exec contenidor-pare -- incus exec client-ubuntu -- dhclient -r eth0
incus exec contenidor-pare -- incus exec client-ubuntu -- dhclient eth0
```

### Problema 2: El client no pot accedir a Internet

**Símptomes:**
```bash
ping 8.8.8.8
# No hi ha resposta
```

**Solucions:**

1. Verifica IP forwarding al router:
```bash
incus exec contenidor-pare -- incus exec router-alpine -- cat /proc/sys/net/ipv4/ip_forward
# Ha de mostrar: 1
```

2. Verifica les regles NAT al router:
```bash
incus exec contenidor-pare -- incus exec router-alpine -- iptables -t nat -L -n -v
# Ha de mostrar la regla MASQUERADE
```

3. Comprova la ruta per defecte al client:
```bash
incus exec contenidor-pare -- incus exec client-ubuntu -- ip route show
# Ha de mostrar: default via 192.168.100.1
```

### Problema 3: El DNS local no funciona

**Símptomes:**
```bash
nslookup router.xarxa.local
# Server failed o NXDOMAIN
```

**Solucions:**

1. Verifica que el client usa el DNS correcte:
```bash
incus exec contenidor-pare -- incus exec client-ubuntu -- cat /etc/resolv.conf
# Ha de mostrar: nameserver 192.168.100.10
```

2. Prova el DNS directament des del servidor:
```bash
incus exec contenidor-pare -- incus exec servidor-ubuntu -- nslookup router.xarxa.local localhost
```

3. Revisa la configuració de dnsmasq:
```bash
incus exec contenidor-pare -- incus exec servidor-ubuntu -- cat /etc/dnsmasq.conf | grep address
```

### Problema 4: /dev/kvm no existeix (virtualització niada no funciona)

**Símptomes:**
```bash
ls -la /dev/kvm
# No such file or directory
```

**Solució:**

Has d'eliminar la VM i crear-la de nou amb virtualització niada:

1. Des de Cloud Shell:
```bash
gcloud compute instances delete ubuntulxd --zone=europe-southwest1-c
```

2. Torna al Pas 8 de la Part 2
3. **Assegura't d'incloure** `--enable-nested-virtualization` a la comanda

### Problema 5: Els contenidors no arrenquen

**Símptomes:**
```bash
incus list
# Mostra contenidors en estat STOPPED o ERROR
```

**Solucions:**

1. Intenta iniciar-los manualment:
```bash
incus start router-alpine
```

2. Revisa els logs:
```bash
incus info router-alpine --show-log
```

3. Si cal, elimina i torna a crear:
```bash
incus delete router-alpine --force
incus launch images:alpine/3.19 router-alpine
```

---

## 📚 Recursos addicionals i documentació

### Documentació oficial

- **Google Cloud Platform**: https://cloud.google.com/docs
  - Compute Engine: https://cloud.google.com/compute/docs
  - Nested virtualization: https://cloud.google.com/compute/docs/instances/nested-virtualization/overview

- **Incus**: https://linuxcontainers.org/incus/docs/
  - Network configuration: https://linuxcontainers.org/incus/docs/main/networks/
  - Container nesting: https://linuxcontainers.org/incus/docs/main/security/#nesting

- **dnsmasq**: https://thekelleys.org.uk/dnsmasq/doc.html
  - Man page: `man dnsmasq`

- **Netplan**: https://netplan.io/
  - Examples: https://netplan.io/examples

- **iptables**: https://www.netfilter.org/
  - NAT tutorial: https://www.karlrupp.net/en/computer/nat_tutorial

### Tutorials relacionats

- **Configuració de NAT amb iptables**: https://www.redswitches.com/blog/iptables-nat/
- **Configuració de DHCP amb dnsmasq**: https://wiki.archlinux.org/title/Dnsmasq
- **Nested containers amb LXD/Incus**: https://ubuntu.com/tutorials/containers-nested

### Llibres recomanats

- "Linux Networking Cookbook" - Carla Schroder
- "TCP/IP Illustrated, Volume 1" - W. Richard Stevens
- "Linux System Administrator's Guide" - Lars Wirzenius

---

## 🎯 Exercicis addicionals per practicar

### Exercici 1: Afegir un segon client

Crea un segon contenidor client (`client-ubuntu-2`) i verifica que també obté IP per DHCP i pot accedir a internet.

**Solució:**
```bash
incus launch images:ubuntu/24.04 client-ubuntu-2 --network incusbr-interna
incus exec client-ubuntu-2 -- bash
# Configura Netplan igual que al primer client
```

### Exercici 2: Configurar un servidor web

Instal·la nginx al `servidor-ubuntu` i configura un domini local `web.xarxa.local` per accedir-hi des del client.

**Pistes:**
- Instal·la nginx: `apt install nginx`
- Afegeix entrada DNS a dnsmasq: `address=/web.xarxa.local/192.168.100.10`
- Accedeix des del client: `curl http://web.xarxa.local`

### Exercici 3: Monitoritzar el tràfic

Instal·la `tcpdump` al router i captura el tràfic entre el client i internet.

**Solució:**
```bash
incus exec router-alpine -- sh
apk add tcpdump
tcpdump -i eth1 -n
# Veuràs tots els paquets que passen pel router
```

### Exercici 4: Crear una tercera xarxa

Crea una nova xarxa interna (`incusbr-dmz` amb rang 192.168.200.0/24) i connecta el servidor web a aquesta xarxa amb el router fent de pont.

### Exercici 5: Configurar reserves DHCP

Configura dnsmasq per assignar sempre la mateixa IP al `client-ubuntu` basant-te en la seua MAC.

**Pista:**
```bash
# Obtenir la MAC del client
incus config get client-ubuntu volatile.eth0.hwaddr

# Afegir a /etc/dnsmasq.conf
dhcp-host=00:16:3e:xx:xx:xx,192.168.100.100,client-ubuntu,infinite
```

---

## ✨ Felicitats!

Has completat la pràctica completa de configuració de xarxes amb Incus a Google Cloud Platform. Ara saps:

✅ Crear i configurar VMs al núvol amb virtualització niada
✅ Utilitzar Cloud Shell per gestionar recursos
✅ Treballar amb contenidors niats
✅ Configurar xarxes complexes amb múltiples nivells
✅ Implementar serveis DHCP i DNS
✅ Configurar enrutament i NAT
✅ Diagnosticar i resoldre problemes de xarxa

**Pròxims passos:**
- Experimenta amb els exercicis addicionals
- Prova altres distribucions Linux (Debian, Fedora, etc.)
- Investiga sobre firewalls amb iptables
- Aprèn sobre VLANs i segmentació de xarxa avançada

🎉 **Bon treball!**

---

## 📝 Notes finals

### Costos de Google Cloud

Recorda que la VM consumeix crèdit mentre està en execució. Per optimitzar:

- **Atura la VM** quan no l'usis: 
  ```bash
  gcloud compute instances stop ubuntulxd --zone=europe-southwest1-c
  ```
- **Monitoritza el consum**: GCP Console → Billing → Overview
- **Elimina recursos** quan acabes: 
  ```bash
  gcloud compute instances delete ubuntulxd --zone=europe-southwest1-c
  ```

### Bones pràctiques

- **Fes snapshots** abans de canvis importants: `incus snapshot contenidor-pare backup1`
- **Documenta els canvis** que fas a la configuració
- **Prova sempre** després de cada canvi de configuració
- **Revisa els logs** quan algo no funciona

### Seguretat

- **No exposes ports** innecessàriament a internet
- **Usa contrasenyes fortes** si actives l'accés remot
- **Actualitza regularment** els contenidors: `apt update && apt upgrade`
- **Monitoritza l'ús** de recursos per detectar anomalies

---

**Data de creació**: Octubre 2024  
**Versió**: 1.0  
**Autor**: Guia creada per a pràctiques educatives  
**Llicència**: Creative Commons BY-NC-SA 4.0