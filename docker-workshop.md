# Installation Docker
```
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo dpkg-reconfigure locales
sudo apt install locales-all
```
```
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo tee /etc/apt/keyrings/docker.asc > /dev/null
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

# Spielen mit Containern
### Einstieg
Hello-World-Container starten:
```
docker run hello-world
```
### Ein kleiner einsamer Container...
Nginx-Container starten:
```
mkdir webdata
docker run -d \
  --name nginx \
  -p 8080:80 \
  -v $(pwd)/webdata:/usr/share/nginx/html \
  nginx

echo "Hallo Azubis" > webdata/index.html
```
--> http://localhost:8080

### ...trifft auf einen zweiten Container
Zweiten Container starten:
```
docker run -it \
  -v $(pwd)/webdata:/data \
  ubuntu bash

echo "<h1>Container 2 war hier</h1>" > /data/index.html
```

### Erreichbarkeit von Containern
```
docker network create appnet
docker run -d \
  --name db \
  --network appnet \
  -e MYSQL_ROOT_PASSWORD=secret \
  mariadb
docker run -it \
  --network appnet \
  ubuntu bash

ubuntu: ping db
ubuntu: apt update && apt install mariadb-client -y
ubuntu: mysql -h db -u root -p
```

### Nginx + Log-Container:
```
docker run -d \
  --name nginx \
  -p 8080:80 \
  -v $(pwd)/logs:/var/log/nginx \
  nginx

docker run -it \
  -v $(pwd)/logs:/logs \
  ubuntu bash

ubuntu: tail -f /logs/access.log
```

### Docker-Compose
```
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./logs:/var/log/nginx

  logger:
    image: alpine
    command: tail -f /logs/access.log
    volumes:
      - ./logs:/logs
```

### 3-Tier Web-App
Aufbau:
```
projekt/
│
├── compose.yaml
│
├── nginx/
│   └── default.conf
│
├── web/
│   ├── index.php
│   └── style.css
│
└── db/
``
```
```
services:
  nginx:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./web:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

  app:
    image: php:8.2-fpm
    volumes:
      - ./web:/var/www/html

  db:
    image: mariadb:latest
    environment:
      MYSQL_ROOT_PASSWORD: geheim
      MYSQL_DATABASE: workshop
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:
```
nginx/default.conf:
```
server {
    listen 80;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \\.php$ {
        fastcgi_pass app:9000;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```
index.php:
```
<?php
echo "<h1>Docker Workshop</h1>";
try {
    $pdo = new PDO(
        "mysql:host=db;dbname=workshop",
        "root",
        "geheim"
    );
    echo "<p>Datenbank verbunden!</p>";

} catch (Exception $e) {
    echo "<p>DB Fehler!</p>";
}
```

```
docker compose up -d
docker compose logs -f
```
http://localhost:8080

### Persistenz:
```
docker compose down
docker compose up -d
```

### Erweiterung:
```
adminer:
    image: adminer
    ports:
      - "8081:8080"
```
localhost:8081

# Gästebuch WebApp
Projektstruktur:
```
projekt/
│
├── docker-compose.yaml
│
├── nginx/
│   └── default.conf
│
├── web/
│   └── index.php
│
└── db/
```

docker-compose.yaml:
```
services:
  nginx:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./web:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

  app:
    image: php:8.2-fpm
    volumes:
      - ./web:/var/www/html

  db:
    image: mariadb:latest
    environment:
      MYSQL_ROOT_PASSWORD: geheim
      MYSQL_DATABASE: gaestebuch
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:
```

nginx/default.conf:
```
server {
    listen 80;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \\.php$ {
        fastcgi_pass app:9000;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

web/index.html:
```
<?php
$host = "db";
$dbname = "gaestebuch";
$user = "root";
$password = "geheim";
try {
    $pdo = new PDO(
        "mysql:host=$host;dbname=$dbname",
        $user,
        $password
    );
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch (PDOException $e) {
    die("DB Verbindung fehlgeschlagen!");
}

# Tabelle erstellen
$pdo->exec("
CREATE TABLE IF NOT EXISTS eintraege (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    nachricht TEXT,
    erstellt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
");

# Formular absenden
if ($_SERVER["REQUEST_METHOD"] === "POST") {
    $name = $_POST["name"];
    $nachricht = $_POST["nachricht"];
    $stmt = $pdo->prepare("
        INSERT INTO eintraege (name, nachricht)
        VALUES (?, ?)
    ");
    $stmt->execute([$name, $nachricht]);
}

# Einträge laden
$eintraege = $pdo->query("
    SELECT * FROM eintraege
    ORDER BY erstellt DESC
");
?>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Docker Gästebuch</title>
    <style>
        body {
            font-family: Arial;
            max-width: 700px;
            margin: auto;
            padding: 20px;
        }
        input, textarea {
            width: 100%;
            margin-bottom: 10px;
            padding: 8px;
        }
        .eintrag {
            border: 1px solid #ccc;
            padding: 10px;
            margin-top: 10px;
        }
    </style>
</head>
<body>
<h1>Docker Gästebuch</h1>
<form method="post">
    <input
        type="text"
        name="name"
        placeholder="Name"
        required
    >
    <textarea
        name="nachricht"
        placeholder="Nachricht"
        required
    ></textarea>
    <button type="submit">
        Absenden
    </button>
</form>
<h2>Einträge</h2>
<?php foreach ($eintraege as $eintrag): ?>
<div class="eintrag">
    <strong>
        <?= htmlspecialchars($eintrag["name"]) ?>
    </strong>
    <p>
        <?= htmlspecialchars($eintrag["nachricht"]) ?>
    </p>
    <small>
        <?= $eintrag["erstellt"] ?>
    </small>
</div>
<?php endforeach; ?>
</body>
</html>
```

```
docker compose up -d
```
http://localhost:8080
