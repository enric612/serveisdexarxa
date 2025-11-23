# Fonaments d'Arquitectura i Configuració del Servidor Web Nginx

## 1. Introducció a l'Arquitectura Nginx
Nginx és un servidor web d'alt rendiment, proxy invers i balancejador de càrrega. A diferència de servidors web tradicionals basats en fils o processos per a cada connexió (com models antics d'Apache), Nginx utilitza una **arquitectura dirigida per esdeveniments (event-driven)** i asíncrona.

Això significa que un procés mestre controla diversos processos de treball (*workers*), i cada *worker* pot gestionar milers de connexions simultànies dins d'un sol fil d'execució, optimitzant l'ús de memòria i CPU sota càrregues elevades.

## 2. Sistema de Configuració i Contextos
La configuració de Nginx és jeràrquica i es basa en **directives** organitzades en **blocs** (o contextos). Entendre l'herència entre aquests blocs és fonamental per a una correcta administració.

### 2.1. Estructura de Fitxers en sistemes Debian/Ubuntu
Encara que la configuració pot residir en un sol fitxer (`nginx.conf`), en entorns de producció s'utilitza una estructura modular per facilitar el manteniment:

*   **Configuració global:** Defineix paràmetres que afecten tot el servei (usuari del procés, nombre de *workers*, rutes de logs).
*   **Sites-Available:** Directori d'emmagatzematge de configuracions de llocs web (Virtual Hosts) que estan disponibles però no necessàriament actius.
*   **Sites-Enabled:** Directori que conté **enllaços simbòlics** apuntant als fitxers de *sites-available*. Nginx només carrega les configuracions presents en aquest directori durant l'arrencada o recàrrega.

### 2.2. La Jerarquia de Contextos
Les directives s'apliquen segons el context on es defineixen:
1.  **Main:** Context global.
2.  **HTTP:** Gestiona la configuració per al trànsit web.
3.  **Server:** Defineix un lloc web o domini virtual específic.
4.  **Location:** Defineix el comportament per a un grup d'URI específic (rutes o carpetes dins del web).

Una directiva definida en un context superior (ex: *http*) es propaga als inferiors (ex: *server*), a menys que sigui sobreescrita explícitament.

## 3. Allotjament Virtual (Virtual Hosting)
El concepte de *Virtual Hosting* permet a un únic servidor web allotjar múltiples llocs web o dominis utilitzant una única adreça IP.

A Nginx, això es gestiona mitjançant els blocs `server`. El mecanisme de resolució funciona de la següent manera:
1.  Nginx rep una petició HTTP.
2.  Inspecciona la capçalera **Host** de la petició (que conté el nom de domini sol·licitat pel client).
3.  Compara aquest valor amb la directiva `server_name` definida en els blocs `server` actius.
4.  Enruta la petició al bloc que coincideix. Si no hi ha coincidència, utilitza el bloc marcat com a `default_server`.

## 4. Nginx com a Proxy Invers i Balancejador de Càrrega
Una de les funcions més potents de Nginx és actuar com a intermediari entre el client i un o més servidors de backend (servidors d'aplicacions).

### 4.1. El Proxy Invers
En aquest model, Nginx accepta la petició, l'envia al servidor d'aplicació (que pot estar en un altre port o màquina) i retorna la resposta al client. Per al client, l'origen sembla ser Nginx.

*   **Directiva `proxy_pass`:** És la instrucció principal que reenvia la petició a l'URL especificat.
*   **Gestió de Capçaleres:** Quan Nginx fa de proxy, la informació original del client (com la seva IP) es perd, ja que el backend veu la petició com si vingués de Nginx. Per solucionar-ho, s'han de redefinir les capçaleres HTTP (ex: `X-Forwarded-For` o `X-Real-IP`) perquè el backend conegui l'origen real.

### 4.2. Upstreams i Balanceig
El mòdul `upstream` permet definir un grup de servidors backend com una sola entitat. Nginx distribueix les peticions entrants entre aquests servidors segons un algoritme:
*   **Round Robin:** Distribució seqüencial (per defecte).
*   **Least Connections:** Envia la petició al servidor amb menys connexions actives.
*   **IP Hash:** Assegura que un mateix client sempre connecti al mateix backend (persistència).

Aquesta arquitectura proporciona alta disponibilitat: si un backend cau, Nginx deixa d'enviar-li trànsit automàticament.

## 5. Seguretat i Xifratge (HTTPS)
La implementació de HTTPS es basa en el protocol TLS (Transport Layer Security). Nginx gestiona la "terminació SSL", la qual cosa significa que desxifra les dades entrants i les xifra a la sortida, alliberant els servidors backend d'aquesta tasca.

Per habilitar HTTPS es requereix:
1.  Un **Certificat Públic** i una **Clau Privada**.
2.  Un bloc `server` escoltant al port 443.
3.  Especificació de protocols segurs (desactivant versions obsoletes com SSLv3 o TLS 1.0).

Addicionalment, és una bona pràctica de seguretat forçar la redirecció de trànsit HTTP (port 80) cap a HTTPS mitjançant codis d'estat 301 (Redirecció permanent).

## 6. Control d'Accés i Gestió d'Errors
Més enllà de l'enrutament, el servidor web actua com a guardià del contingut.

### 6.1. Autenticació HTTP Bàsica
El protocol HTTP inclou un mecanisme simple d'autenticació on el navegador sol·licita un usuari i contrasenya i els envia codificats en Base64 a la capçalera `Authorization`. Nginx valida aquestes credencials contra un fitxer local (habitualment generat amb `htpasswd`). Això permet protegir rutes específiques (blocs `location`) sense necessitat de programació en el backend.

### 6.2. Pàgines d'Error Personalitzades
Quan el servidor troba un problema (recurs no trobat - 404, error intern - 500), retorna una pàgina HTML per defecte. Mitjançant la directiva `error_page`, és possible interceptar aquests codis d'estat i servir fitxers estàtics personalitzats, millorant l'experiència d'usuari i evitant exposar informació tècnica del servidor.
