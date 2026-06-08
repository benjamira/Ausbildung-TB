# Juice-Shop starten

```
services:
  juice-shop:
    image: bkimminich/juice-shop:v19.2.1
    container_name: juice-shop
    restart: unless-stopped
    ports:
      - 3000:3000
```

