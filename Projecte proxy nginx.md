# Activitats Pràctiques amb Nginx sobre Ubuntu

A continuació es presenten 3 propostes d'activitats pràctiques per treballar amb Nginx, ordenades per nivell de dificultat (bàsic, intermedi i avançat).

---

## Activitat 1: Desplegament de Virtual Hosts (Server Blocks)

**Nivell:** Bàsic

### Objectiu
Configurar un únic servidor Ubuntu perquè allotgi dos llocs web independents utilitzant noms de domini simulats.

### Escenari
Ets l'administrador de sistemes d'una petita agència web. Has d'allotjar el lloc web d'un "Client A" i el d'un "Client B" en el mateix servidor per estalviar costos.

### Requisits

1.  **Instal·lació:** Instal·lar el servidor web Nginx a Ubuntu.
2.  **Estructura de Directoris:** Crear dues carpetes separades dins de `/var/www/` (per exemple, `client_a` i `client_b`).
3.  **Contingut:** Crear un fitxer `index.html` diferent per a cada carpeta que identifiqui clarament quin lloc s'està visitant (Ex: "Benvinguts al Client A").
4.  **Permisos:** Assegurar que l'usuari propietari dels fitxers sigui el correcte (habitualment `www-data` o l'usuari actual) i els permisos siguin segurs.
5.  **DNS Local:** Modificar el fitxer `/etc/hosts` de la teva màquina (client) per apuntar els dominis `www.clienta.test` i `www.clientb.test` a la IP del servidor Ubuntu.
6.  **Configuració Nginx:**
    *   Crear dos arxius de configuració de bloc de servidor (Server Blocks) a `/etc/nginx/sites-available/`.
    *   Activar-los creant enllaços simbòlics a `/etc/nginx/sites-enabled/`.
7.  **Verificació:**
    *   En entrar a `http://www.clienta.test` s'ha de veure el web del Client A.
    *   En entrar a `http://www.clientb.test` s'ha de veure el web del Client B.
    *   Si s'accedeix per la IP directament, hauria de mostrar un error o la pàgina per defecte d'Nginx.

---

## Activitat 2: Proxy Invers i Balanceig de Càrrega

**Nivell:** Intermedi

### Objectiu
Utilitzar Nginx com a intermediari per distribuir el trànsit entre diverses instàncies d'una aplicació (backend).

### Escenari
Tens una aplicació web que rep moltes visites. Per evitar que el servidor caigui, has aixecat dues instàncies lleugeres del servei (backends) en ports diferents i vols que Nginx reparteixi les peticions entre elles.

### Requisits

1.  **Preparació del Backend:** Simula dos servidors de backend escoltant en ports diferents (per exemple, port `8081` i `8082`).
    > *Nota: Pots fer servir `python3 -m http.server [port]` en dues terminals diferents amb contingut diferent per distingir-los.*
2.  **Configuració Upstream:** Definir un grup `upstream` a la configuració d'Nginx que inclogui els dos servidors locals (`localhost:8081` i `localhost:8082`).
3.  **Proxy Pass:** Configurar un `location /` dins del bloc de servidor principal que reenviï tot el trànsit al grup `upstream` definit anteriorment.
4.  **Algoritme de Balanceig:** Configurar l'algoritme de balanceig perquè sigui *Round Robin* (per defecte) o *Least Connections*.
5.  **Capçaleres (Headers):** Configurar Nginx perquè passi la IP real del client al backend utilitzant `X-Real-IP` o `X-Forwarded-For`.
6.  **Verificació:** En refrescar repetidament la pàgina principal del servidor Nginx (port 80), la resposta ha d'alternar entre el contingut del backend 1 i el backend 2.
7.  **Tolerància a fallades:** Aturar un dels backends simulats i verificar que Nginx deixa d'enviar trànsit a aquest node i el web segueix funcionant amb l'altre.

---

## Activitat 3: Seguretat, HTTPS (SSL) i Restricció d'Accés

**Nivell:** Avançat

### Objectiu
Securitzar un servidor web implementant xifratge SSL, autenticació bàsica i pàgines d'error personalitzades.

### Escenari
Has de protegir un panell d'administració intern. L'accés ha de ser xifrat, requerir contrasenya i els errors han de ser amigables.

### Requisits

1.  **Certificats:** Generar un certificat autofirmat (OpenSSL) vàlid per a 365 dies.
2.  **Configuració SSL:** Configurar Nginx per escoltar al port `443` (HTTPS) utilitzant el certificat i la clau privada generats.
3.  **Redirecció Forçada:** Configurar el servidor perquè tot el trànsit que arribi pel port `80` (HTTP) es redirigeixi automàticament (Codi 301) al port `443` (HTTPS).
4.  **Optimització de Seguretat:**
    *   Desactivar protocols antics (com SSLv3 o TLS 1.0/1.1) i permetre només TLS 1.2 o superior.
    *   Ocultar la versió de Nginx a les capçaleres de resposta utilitzant `server_tokens`.
5.  **Autenticació Bàsica:** Protegir una ruta específica (per exemple, `/admin`) de manera que sol·liciti usuari i contrasenya utilitzant un fitxer `.htpasswd`.
6.  **Pàgines d'Error:** Crear una pàgina HTML personalitzada per a l'error **404 (Not Found)** i configurar Nginx perquè la mostri en lloc de la pàgina per defecte quan es demani un fitxer inexistent.
7.  **Verificació:**
    *   Accedir per HTTP i veure com canvia automàticament a HTTPS.
    *   El navegador mostrarà una advertència de seguretat (normal en certificats autofirmats); s'ha d'acceptar.
    *   En entrar a `/admin`, el navegador ha de demanar usuari i contrasenya.
    *   En entrar a `/no-existeix`, ha de sortir la teva pàgina d'error personalitzada.
