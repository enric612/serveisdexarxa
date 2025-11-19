# Unitat Didàctica: Proxy Invers i Balanceig amb Nginx (Entorn Remot)

## PART 1: INTRODUCCIÓ TEÒRICA

### 1. Què és realment un Proxy?

Imagineu que voleu demanar un llibre a una biblioteca molt exclusiva on no deixen entrar al públic general. Vosaltres (el **Client**) no podeu parlar directament amb el bibliotecari (el **Servidor**).

Què feu? Contracteu un assistent. Vosaltres li doneu el títol del llibre a l'assistent, ell entra a la biblioteca, agafa el llibre i us el porta.
*   El bibliotecari només ha vist l'assistent (no sap qui sou vosaltres).
*   Vosaltres heu aconseguit el llibre sense entrar.

En informàtica, aquest assistent és el **Proxy**. És un intermediari entre un usuari i internet (o un servidor).

### 2. Tipus de Proxy: Forward vs Reverse

Encara que la paraula és la mateixa, fan funcions oposades:

1.  **Forward Proxy (Proxy d'encaminament):** L'utilitza l'usuari per sortir a Internet.
    *   *Exemple:* Una escola que bloqueja xarxes socials. Els alumnes passen pel proxy, i aquest diu "Prohibit". Protegeix la sortida.
2.  **Reverse Proxy (Proxy Invers):** L'utilitza el servidor per rebre peticions d'Internet. **(Aquest és el nostre cas avui)**.
    *   *Exemple:* Quan entreu a Google, no us connecteu al seu servidor final. Us connecteu al seu Proxy Invers, i ell decideix a quin ordinador intern envia la vostra petició.

### 3. Per què utilitzar un Proxy Invers (Nginx)?

Per què no ens connectem directament al servidor de l'aplicació?

*   **Seguretat:** Amaga la identitat i la IP real dels servidors interns. Els hackers només veuen el proxy.
*   **Balanceig de càrrega (Load Balancing):** Si tens 3 servidors amb la mateixa web, el proxy reparteix les visites perquè cap es col·lapsi.
*   **Rendiment:** Pot guardar còpies de la web (caché) per servir-les més ràpid.

---

## PART 2: LABORATORI PRÀCTIC

> **⚠ NOTA IMPORTANT SOBRE L'ENTORN**
> Esteu connectats a una **màquina remota**. Com que **no sou administradors del vostre PC local**, no podem modificar els fitxers del sistema per inventar-nos un nom de domini (com `www.la-meva-web.com`).
>
> Per tant:
> 1. Treballareu tot des de la terminal de la màquina remota.
> 2. Per veure el resultat al navegador, utilitzareu l'**Adreça IP** de la màquina remota.

### Pas 1: Identificar la nostra màquina
Com has entrat per ssh ja la saps, per tant tin en compte aquesta direcció per entrar despres via web.

---

### Pas 2: Simular l'aplicació interna (Backend)
Simularem que tenim 3 aplicacions funcionant alhora dins del servidor. Com que no necessitem saber programar, utilitzarem Python per crear servidors web instantanis.

NOTA: Tranquil no em poses cap queixa, no vas a programar en python, ni que fores informàtic.

**1. Crear el contingut web**
Executa aquestes comandes per crear 3 carpetes i 3 fitxers HTML diferents:

```bash
# Creem les carpetes
mkdir web1 web2 web3

# Creem el contingut per diferenciar els servidors
echo "<h1>Hola! Sóc el SERVIDOR 1 (Port 3000)</h1>" > web1/index.html
echo "<h1>Hola! Sóc el SERVIDOR 2 (Port 3001)</h1>" > web2/index.html
echo "<h1>Hola! Sóc el SERVIDOR 3 (Port 3002)</h1>" > web3/index.html
```

**2. Activar els servidors**
Ara hem de posar en marxa aquestes webs. Necessitaràs obrir **3 terminals noves** (o pestanyes de terminal), ja que cada servidor ocupa una finestra.

*   **Terminal 1:**
    ```bash
    cd web1
    python3 -m http.server 3000
    ```
*   **Terminal 2:**
    ```bash
    cd web2
    python3 -m http.server 3001
    ```
*   **Terminal 3:**
    ```bash
    cd web3
    python3 -m http.server 3002
    ```

*Ara tenim 3 webs funcionant als ports 3000, 3001 i 3002. No tanquis aquestes terminals.*

*Segurament sabras accedir obrint 3 sessions ssh pero la versio pro seria utilitzar una eina com termmux, eixa part la deixe per als pros de classe, pregunteuli al vostre amic gpt que vos ajudara (o proveu el nou gemini 3 a Google Studio AI.*

---

### Pas 3: Instal·lació i Configuració del Proxy
Ara mateix, per veure aquestes webs hauríeu d'escriure la IP seguida del port (ex: `192.168.1.50:3000`). Volem que l'usuari entri normalment (Port 80) i Nginx s'encarregui de tot.

**Obre una 4a terminal per treballar:**

1.  **Instal·lar Nginx:**
    ```bash
    sudo apt update
    sudo apt install nginx -y
    ```

2.  **Netejar configuracions prèvies:**
    Esborrem la web per defecte perquè no ens molesti:
    ```bash
    sudo rm /etc/nginx/sites-enabled/default
    ```

3.  **Crear la configuració del Proxy:**
    ```bash
    sudo nano /etc/nginx/sites-available/laboratori
    ```

4.  **Configuració (Copia i Enganxa):**
    Fixa't en `server_name _`. Això és un comodí. Li diu a Nginx: "Accepta qualsevol petició que arribi a aquest servidor, vingui del domini que vingui, o directament de la IP".

    ```nginx
    server {
        listen 80;
        # El guió baix fa que respongui a qualsevol nom o IP
        server_name _;

        location / {
            # Redirigim tot el trànsit al servidor intern 1
            proxy_pass http://127.0.0.1:3000;
            
            # Capçaleres importants
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
    ```

5.  **Activar la web i Reiniciar:**
    ```bash
    sudo ln -s /etc/nginx/sites-available/laboratori /etc/nginx/sites-enabled/
    sudo systemctl restart nginx
    ```

**PROVA AL NAVEGADOR:**
1.  Obre el navegador del teu PC local (Chrome/Firefox).
2.  Escriu l'**Adreça IP** que has apuntat al Pas 1 (ex: `http://192.168.1.XX`).
3.  Hauries de veure: **"Hola! Sóc el SERVIDOR 1 (Port 3000)"**.

---

### Pas 4: Configuració del Balanceig de Càrrega

Ara farem que Nginx reparteixi la feina entre els 3 servidors que hem creat, perquè si un cau, els altres segueixin funcionant.

1.  **Editem de nou la configuració:**
    ```bash
    sudo nano /etc/nginx/sites-available/laboratori
    ```

2.  **Canviem el fitxer per aquest:**
    Afegim el bloc `upstream` al principi.

    ```nginx
    # Definim el nostre grup de servidors
    upstream backend_cluster {
        server 127.0.0.1:3000;
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
    }

    server {
        listen 80;
        server_name _;

        location / {
            # Ara enviem el trànsit al GRUP
            proxy_pass http://backend_cluster;
            
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
    ```

3.  **Guardar i reiniciar:**
    Prem `Ctrl+O`, `Enter`, `Ctrl+X`. Després reinicia:
    ```bash
    sudo systemctl restart nginx
    ```

---

### Pas 5: Comprovació Final

Tenim dues formes de comprovar que funciona:

**A) Des del Navegador del teu PC:**
1.  Entra a la IP de la màquina remota.
2.  Refresca la pàgina (F5) ràpidament diverses vegades.
3.  Veuràs com el missatge canvia: *Servidor 1 -> Servidor 2 -> Servidor 3...*

**B) Des de la terminal (Mode "Hacker"):**
Si volem simular que tenim un domini sense tenir-ne un, podem utilitzar l'eina `curl` des de la mateixa terminal remota. `curl` és com un navegador però només de text.

Escriu això a la terminal:
```bash
curl http://localhost
```

*(Veuràs el codi HTML d'un dels servidors).*

Repeteix la comanda diverses vegades. Veuràs com cada vegada respon un servidor diferent. Nginx està balancejant la càrrega perfectament!

---

Aquesta arquitectura és la base de com funcionen serveis com Netflix, Google o Amazon per atendre milions d'usuaris alhora.
