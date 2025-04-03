# firebird-cheat-sheet

# GUI
- dbeaver
  
- https://github.com/mariuz/flamerobin/releases <-- Only works with local files?




<br><br>
---
<br><br>



#  Docker-Compose 

<details><summary>Click to expand..</summary>

Following commands work for 2.5 & 5.0:

## 🔍 Container Logs prüfen  
Enthält auch die Firebird-Version:  
```shell
docker logs dev-environment-firebird-1
```

## 🔄 Docker Container Neustart  
```shell
docker restart dev-environment-firebird-1
```

---

## 🔧 Konfigurations- und Datenbankprüfung  
### 📜 Prüfen, ob die Config existiert  
```shell
docker exec -it dev-environment-firebird-1 cat /firebird/etc/firebird.conf
```

### 📂 Volumes und Pfade sicherstellen  
```shell
docker exec -it dev-environment-firebird-1 ls -l /firebird/etc/
```

### 🗄️ Prüfen, ob die Datenbank existiert  
```shell
docker exec -it dev-environment-firebird-1 ls -l /firebird/data/
```

---

# 🔥 Firebird 2.5.8-ss  
🔗 [Docker Hub](https://hub.docker.com/layers/jacobalberty/firebird/2.5.8-ss/images/sha256-0749a634c0fed18ef60ad18e0634d9f48822ab7a7aab2f708630288ad96f48f1)  

### 🛠️ Docker-Compose  
```yaml
version: '3.8'

services:
  firebird:
    image: jacobalberty/firebird:2.5.8-ss
    environment:
      - ISC_PASSWORD=masterkey
      - FIREBIRD_USER=test
      - FIREBIRD_PASSWORD=test
      - FIREBIRD_DATABASE=testdb.fdb
      - EnableLegacyClientAuth=true
      - EnableWireCrypt=false
      - DataTypeCompatibility=2.5
      - TZ=Europe/Berlin
    ports:
      - "3050:3050"
    volumes:
      - ${FIREBIRD_HOME:-~/data/firebird}/data:/firebird/data
      - ${FIREBIRD_HOME:-~/data/firebird}/system:/firebird/system
      - ${FIREBIRD_HOME:-~/data/firebird}/etc:/firebird/etc
    healthcheck:
      test: ["CMD-SHELL", "nc -z localhost 3050"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## 🖥️ CLI-Kommandos (Firebird 2.5)  
<details><summary>▶️ Datenbankverbindung</summary>

```shell
# Als SYSDBA
docker exec -it dev-environment-firebird-1 /usr/local/firebird/bin/isql -user SYSDBA -password masterkey localhost:/firebird/data/testdb.fdb

# Als normaler Benutzer
docker exec -it dev-environment-firebird-1 /usr/local/firebird/bin/isql -user test -password test localhost:/firebird/data/testdb.fdb

# Sobald du verbunden bist:
SHOW DATABASE;
```
</details>

<details><summary>▶️ SYSDBA Passwort prüfen/ändern</summary>

```shell
# Passwort anzeigen
docker exec -it dev-environment-firebird-1 cat /firebird/etc/SYSDBA.password

# Passwort ändern (gsec utility)
docker exec -it dev-environment-firebird-1 /usr/local/firebird/bin/gsec -user SYSDBA -password masterkey
```
</details>

<details><summary>▶️ Firebird Logs ansehen</summary>

```shell
docker exec -it dev-environment-firebird-1 cat /firebird/log/firebird.log
```
</details>

---

## 🔄 Firebird 5.0  
🔗 [Docker Hub](https://hub.docker.com/r/firebirdsql/firebird)  

### 🛠️ Docker-Compose  
```yaml
version: '3.8'

services:
  firebird:
    image: firebirdsql/firebird:latest
    environment:
      - FIREBIRD_ROOT_PASSWORD=masterkey
      - FIREBIRD_USER=test
      - FIREBIRD_PASSWORD=test
      - FIREBIRD_DATABASE=testdb.fdb
      - FIREBIRD_DATABASE_DEFAULT_CHARSET=UTF8
      - TZ=Europe/Berlin
      - FIREBIRD_CONF_WireCrypt=Disabled
    ports:
      - "3050:3050"
    volumes:
      - ${FIREBIRD_HOME:-~/data/firebird}:/var/lib/firebird/data
    healthcheck:
      test: ["CMD", "isql", "-user", "SYSDBA", "-password", "masterkey", "localhost:testdb.fdb"]
      interval: 10s
      timeout: 5s
      retries: 5
```

---

## 🖥️ CLI-Kommandos (Firebird 5.0)  
<details><summary>▶️ Datenbankverbindung</summary>

```shell
docker exec -it dev-environment-firebird-1 isql -user SYSDBA -password masterkey localhost:/var/lib/firebird/data/testdb.fdb

# Sobald du verbunden bist:
SHOW DATABASE;
```
</details>

---

## ⚖️ Unterschiede Firebird 2.5 vs. 5.0  
| Feature                 | Firebird 2.5 (`jacobalberty/firebird`) | Firebird 5.0 (`firebirdsql/firebird`) |
|-------------------------|---------------------------------|----------------------------------|
| **Verzeichnisstruktur** | `/firebird/data` für DBs, `/firebird/etc` für Configs | `/var/lib/firebird/data` für DBs, Configs direkt unter `/etc` |
| **Auth & Passwort**     | `ISC_PASSWORD` für SYSDBA-Passwort | `FIREBIRD_ROOT_PASSWORD` für SYSDBA-Passwort |
| **WireCrypt**           | `EnableWireCrypt=false` nötig für unverschlüsselte Verbindungen | `FIREBIRD_CONF_WireCrypt=Disabled` statt `EnableWireCrypt` |
| **Health Check**        | Prüft mit `nc -z localhost 3050` | Prüft mit `isql`-Kommando |
| **Passwort-Länge**      | Max. 8 Zeichen für Firebird 2.5 | Längere Passwörter erlaubt |

---

## 🛠️ Wichtige Firebird-Pfade  
📂 **Firebird 2.5**  
- **Basis:** `/usr/local/firebird`  
- **Binaries:** `/usr/local/firebird/bin`  
- **Daten:** `/firebird/data`  
- **Konfiguration:** `/firebird/etc`  
- **Logs:** `/firebird/log`  

📂 **Firebird 5.0**  
- **Basis:** `/var/lib/firebird`  
- **Binaries:** `/usr/bin`  
- **Daten:** `/var/lib/firebird/data`  
- **Konfiguration:** `/etc/firebird/`  
- **Logs:** `/var/log/firebird.log`  

---

## 🎯 Tipps für die Nutzung  
✅ **Immer den vollständigen Pfad zu den Binaries angeben** (`/usr/local/firebird/bin/isql` oder `/usr/bin/isql`)  
✅ **SYSDBA Passwort** findet sich in `/firebird/etc/SYSDBA.password` (2.5) oder `/etc/firebird/SYSDBA.password` (5.0)  
✅ **Für node-firebird Verbindungen:** `EnableWireCrypt=false` setzen  


</details>















<br><br>
---
<br><br>



# Node.js

<details><summary>Click to expand..</summary>

# node-firebird
- https://github.com/hgourvest/node-firebird

</details>
