# Docker Compose Seminar

**Praktischer Einstieg in Container-Orchestrierung mit Nginx Load Balancing**

*Schulungsmaterial für Mischok CI*

---

## Überblick & Lernziele

In diesem Seminar lernen Sie, wie Sie mit Docker Compose eine lokale Multi-Container-Umgebung aufbauen und verwalten. Sie werden einen Load Balancer konfigurieren und verschiedene Load-Balancing-Strategien ausprobieren.

### Lernziele

- Docker Images verstehen und Docker Compose verwenden
- Mehrere Container mit Docker Compose orchestrieren
- Nginx als Load Balancer konfigurieren
- Verschiedene Load-Balancing-Methoden ausprobieren
- Development-Setup mit automatischer Synchronisierung

### Docker Compose Architektur

```
┌─────────────────────────────────────────────────┐
│        Docker Compose Architektur               │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │   Nginx Load Balancer (Port 8080)      │   │
│  │   - Round Robin / IP Hash / Least Conn │   │
│  └─────────────────────────────────────────┘   │
│              ↓                 ↓                │
│  ┌──────────────────┐  ┌──────────────────┐   │
│  │  Webserver 1     │  │  Webserver 2     │   │
│  │  Nginx Container │  │  Nginx Container │   │
│  │  Port 8081       │  │  Port 8082       │   │
│  │  index.html v1   │  │  index.html v2   │   │
│  └──────────────────┘  └──────────────────┘   │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Schritt 1: Grundlagen - Nginx Container mit Index.html

Wir beginnen mit einem einfachen Nginx Container, der eine statische HTML-Seite bereitstellt.

### 1.1 Verzeichnisstruktur erstellen

```
webserver1/
└── html/
    └── index.html
```

### 1.2 Index.html erstellen

**webserver1/html/index.html**

```html
<!DOCTYPE html>
<html>
<head>
    <title>Webserver 1</title>
    <style>
        body { font-family: Arial; text-align: center; padding: 50px; }
        .container { background: #4CAF50; color: white; padding: 30px; border-radius: 10px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🟢 Webserver 1</h1>
        <p>Dies ist die erste Instanz</p>
        <p>Hostname: <strong><?php echo gethostname(); ?></strong></p>
    </div>
</body>
</html>
```

### 1.3 Dockerfile erstellen

**webserver1/Dockerfile**

```dockerfile
FROM nginx:latest

# HTML-Dateien in den Nginx Container kopieren
COPY html/ /usr/share/nginx/html/

# Nginx starten
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

> **Tipp:** Das `nginx:latest` Image ist sehr leicht und enthält bereits einen vollständig konfigurierten Webserver.

---

## Schritt 2: Docker Compose einführen

Docker Compose ermöglicht es, mehrere Container als Dienst zu definieren und zu verwalten.

### 2.1 compose.yaml erstellen

**compose.yaml**

```yaml
services:
  webserver1:
    build:
      context: ./webserver1
    ports:
      - "8081:80"
    container_name: webserver1
```

### 2.2 Häufige Docker Compose Kommandos

| Kommando | Beschreibung |
|----------|-------------|
| `docker compose up` | Container starten und im Vordergrund anzeigen |
| `docker compose up -d` | Container im Hintergrund starten |
| `docker compose down` | Container stoppen und entfernen |
| `docker compose logs` | Logs aller Container anzeigen |
| `docker compose ps` | Status der Container anzeigen |

### 2.3 Test

```bash
$ docker compose up
$ curl http://localhost:8081
```

✓ Sie sollten jetzt die HTML-Seite von Webserver 1 sehen können

---

## Schritt 3: Zweiter Webserver hinzufügen

Wir erstellen einen zweiten Container mit einer abgeänderten index.html.

### 3.1 Struktur für webserver2

```
webserver2/

├── Dockerfile

└── html/

    └── index.html
```

### 3.2 Index.html für Webserver 2

**webserver2/html/index.html**

```html
<!DOCTYPE html>
<html>
<head>
    <title>Webserver 2</title>
    <style>
        body { font-family: Arial; text-align: center; padding: 50px; }
        .container { background: #2196F3; color: white; padding: 30px; border-radius: 10px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🔵 Webserver 2</h1>
        <p>Dies ist die zweite Instanz</p>
        <p>Hostname: <strong><?php echo gethostname(); ?></strong></p>
    </div>
</body>
</html>
```

### 3.3 Docker Compose aktualisieren

**compose.yaml**

```yaml
services:
  webserver1:
    build:
      context: ./webserver1
    ports:
      - "8081:80"
    container_name: webserver1

  webserver2:
    build:
      context: ./webserver2
    ports:
      - "8082:80"
    container_name: webserver2
```

### 3.4 Test mit beiden Servern

```bash
$ docker compose down
$ docker compose up -d
$ curl http://localhost:8081  # Webserver 1
$ curl http://localhost:8082  # Webserver 2
```

✓ Sie sehen unterschiedliche Seiten (grün und blau) je nachdem, welchen Port Sie nutzen

---

## Schritt 4: Nginx Load Balancer konfigurieren

Ein Nginx Container leitet Requests im Round-Robin-Verfahren an die beiden Webserver weiter.

### 4.1 Load Balancer Verzeichnisstruktur

```
loadbalancer/
├── Dockerfile
└── nginx.conf
```

### 4.2 Nginx.conf - Round Robin (Standard)

**loadbalancer/nginx.conf**

```nginx
http {
    upstream backend {
        server webserver1:80;
        server webserver2:80;
    }

    server {
        listen 80;
        server_name _;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

### 4.3 Dockerfile für Load Balancer

**loadbalancer/Dockerfile**

```dockerfile
FROM nginx:latest

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 4.4 Docker Compose mit Load Balancer

**compose.yaml**

```yaml
services:
  loadbalancer:
    build:
      context: ./loadbalancer
    ports:
      - "8080:80"
    depends_on:
      - webserver1
      - webserver2
    container_name: loadbalancer

  webserver1:
    build:
      context: ./webserver1
    container_name: webserver1

  webserver2:
    build:
      context: ./webserver2
    container_name: webserver2
```

### 4.5 Test des Load Balancers

```bash
$ docker compose up -d
$ curl http://localhost:8080  # Zeigt Webserver 1
$ curl http://localhost:8080  # Zeigt Webserver 2
$ curl http://localhost:8080  # Zeigt Webserver 1 (wieder)
```

✓ Sie sehen abwechselnd grüne und blaue Seiten - das ist Round Robin!

---

## Schritt 5: Load Balancing Methoden

Nginx unterstützt verschiedene Methoden zur Verteilung von Anfragen.

### 5.1 IP Hash - Sticky Sessions

Jede Client-IP wird immer zum gleichen Backend-Server geleitet.

**loadbalancer/nginx.conf (IP Hash Variante)**

```nginx
upstream backend {
    ip_hash;
    server webserver1:80;
    server webserver2:80;
}
```

**Verwendungsfall:** Sessions müssen auf demselben Server bleiben

### 5.2 Least Connections - Least Conn

Anfragen werden an den Server mit den wenigsten aktiven Verbindungen geleitet.

**loadbalancer/nginx.conf (Least Conn Variante)**

```nginx
upstream backend {
    least_conn;
    server webserver1:80;
    server webserver2:80;
}
```

**Verwendungsfall:** Server haben unterschiedliche Leistungsprofile

### 5.3 Weighted Round Robin

Verschiedene Gewichtungen für unterschiedliche Server-Kapazitäten.

```nginx
upstream backend {
    server webserver1:80 weight=3;  # 3x mehr Traffic
    server webserver2:80 weight=1;  # 1x Traffic
}
```

### 5.4 Methoden testen

**Aufgabe: Verschiedene Methoden testen**

1. Ändern Sie `nginx.conf` zu `ip_hash;`
2. Führen Sie aus: `docker compose restart loadbalancer`
3. Machen Sie mehrere Requests: `for i in {1..10}; do curl http://localhost:8080; done`
4. Beobachten Sie, dass Sie immer denselben Server bekommen
5. Probieren Sie `least_conn;` aus

> **Wichtig:** Nach Änderungen an nginx.conf müssen Sie den Load Balancer neu starten oder `docker compose down && docker compose up` ausführen.

---

## Schritt 6: Finale Konfiguration und Tests

Die komplette Lösung mit Load Balancer in Aktion.

### 6.1 Finale compose.yaml

Die fertige Konfiguration mit allen Services:

**compose.yaml**

```yaml
services:
  loadbalancer:
    build:
      context: ./loadbalancer
    ports:
      - "8080:80"
    depends_on:
      - webserver1
      - webserver2
    container_name: loadbalancer

  webserver1:
    build:
      context: ./webserver1
    container_name: webserver1

  webserver2:
    build:
      context: ./webserver2
    container_name: webserver2
```

### 6.2 System starten und testen

```bash
$ docker compose up -d
$ docker compose ps            # Status prüfen
$ curl http://localhost:8080   # Zeigt Webserver 1
$ curl http://localhost:8080   # Zeigt Webserver 2
$ curl http://localhost:8080   # Zeigt Webserver 1 (wieder)
```

✓ Sie sehen abwechselnd grüne und blaue Seiten - Round Robin funktioniert!

### 6.3 Logs überwachen

Mit dem folgenden Kommando sehen Sie live, wie Requests verteilt werden:

```bash
$ docker compose logs -f loadbalancer
```

### 6.4 Praktische Übungen

**Aufgabe: Load Balancer in Aktion beobachten**

1. Terminal 1: `docker compose logs -f loadbalancer`
2. Terminal 2: `for i in {1..10}; do curl http://localhost:8080; sleep 0.5; done`
3. Beobachten Sie, wie die Requests abwechselnd verteilt werden
4. Stoppen Sie einen Server: `docker compose stop webserver1`
5. Machen Sie neue Requests - sie gehen nur noch zu webserver2
6. Starten Sie webserver1 wieder: `docker compose start webserver1`

---

## Best Practices & Tipps

### Netzwerk Debugging

```bash
# In einen laufenden Container gehen
docker exec -it webserver1 bash

# Ping zwischen Containern testen
ping webserver2

# Logs eines Containers ansehen
docker compose logs loadbalancer -f
```

### Container-Lifecycle verwalten

| Szenario | Kommando |
|----------|----------|
| Nur Logs überprüfen | `docker compose logs -f` |
| Container neu bauen | `docker compose build --no-cache` |
| Einen Service neu starten | `docker compose restart loadbalancer` |
| Alle Ressourcen entfernen | `docker compose down -v` |

### Performance Tipps

- **Multi-stage Builds:** Reduzieren Sie Image-Größe mit mehrstufigen Dockerfiles
- **Cache Layer:** Häufig geänderte Commands am Ende platzieren
- **Memory Limits:** Setzen Sie Memory-Limits für unkontrolliertes Wachstum
- **Health Checks:** Definieren Sie Health Checks für automatisches Restart bei Fehlern

**Health Check Beispiel in compose.yaml:**

```yaml
services:
  webserver1:
    build: ./webserver1
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## Zusammenfassung der wichtigsten Konzepte

### Was haben wir gelernt?

- **Docker Images:** Basis für Container mit Applikationen
- **Docker Compose:** Vereinfacht Multi-Container-Verwaltung
- **Service-Discovery:** Container finden sich automatisch
- **Load Balancing:** Verteilt Traffic auf mehrere Backend-Server
- **Verschiedene Load-Balancing-Methoden:** Round Robin, IP Hash, Least Conn
- **Nginx als Reverse Proxy:** Professionelle Last-Verteilung

### Zusammenfassung: Finale compose.yaml

**compose.yaml**

```yaml
services:
  loadbalancer:
    build:
      context: ./loadbalancer
    ports:
      - "8080:80"
    depends_on:
      - webserver1
      - webserver2
    container_name: loadbalancer

  webserver1:
    build:
      context: ./webserver1
    container_name: webserver1

  webserver2:
    build:
      context: ./webserver2
    container_name: webserver2
```

### Nächste Schritte

- Erweitern Sie das Setup mit Datenbanken (PostgreSQL, MySQL, Redis)
- Implementieren Sie Health Checks für automatisches Container-Restart
- Fügen Sie Volumes für persistente Daten hinzu
- Nutzen Sie Docker Registries für Image Distribution
- Erforschen Sie Kubernetes für Production Deployments
- Automatisieren Sie Deployments mit CI/CD Pipelines

---

## Troubleshooting

### Häufige Fehler und Lösungen

#### Problem: "Address already in use"

**Lösung:** Port wird bereits verwendet

```bash
# Laufende Container auflisten
docker ps

# Oder Port ändern in compose.yaml
ports:
  - "9090:80"  # Statt 8080
```

#### Problem: "Container exits immediately"

**Lösung:** Logs prüfen

```bash
docker compose logs [service-name]
docker compose logs --follow  # Live-Logs
```

#### Problem: "Cannot reach container from another"

**Lösung:** Container-Namen und Portqualität überprüfen

```bash
docker compose exec webserver1 ping webserver2
docker compose ps
docker compose logs loadbalancer
```

#### Problem: "Load Balancer verteilt nicht korrekt"

**Lösung:** nginx.conf überprüfen und Service neu starten

```bash
docker compose logs -f loadbalancer
docker compose restart loadbalancer
docker compose up -d --build loadbalancer
```

---

## Weitere Ressourcen

### Offizielle Dokumentation

- **Docker Compose:** https://docs.docker.com/compose/
- **Nginx Image:** https://hub.docker.com/_/nginx
- **Docker Networking:** https://docs.docker.com/network/

### Empfohlene Tools

- **Docker Desktop:** IDE für Docker (Mac, Windows, Linux)
- **Portainer:** Web-UI für Docker Management
- **ctop:** Top-ähnliche Überwachung für Container

### Verwandte Technologien

- **Kubernetes:** Enterprise Container Orchestration
- **Docker Swarm:** Clustering mit Docker
- **Docker Registry:** Private Image Repositories

---

**Docker Compose Seminar - Mischok CI | Erstellt am 16. Juni 2026**

Dieses Handout darf frei verwendet und weitergegeben werden.
