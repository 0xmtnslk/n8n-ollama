## 1. Sistem Güncelleme ve Temel Paketler
```
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git ufw nginx certbot python3-certbot-nginx postgresql postgresql-contrib
```


## 2. Güvenlik - UFW Firewall Ayarları

```
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'  # 80 ve 443 portları için
sudo ufw enable
sudo ufw status
```
## 3. PostgreSQL Kurulumu (n8n için önerilir)
```
sudo -u postgres createuser --interactive
# Kullanıcı adı: n8nuser
# Superuser? hayır
# Shall the new role be allowed to create databases? (y/n) farklı databaseler oluşturacaksa evet yoksa hayır
# Shall the new role be allowed to create more new roles? (y/n)  buna hayır diyoruz.
sudo -u postgres createdb n8ndb -O n8nuser
sudo -u postgres psql
# Şifre belirle:
ALTER USER n8nuser WITH PASSWORD 'GüçlüBirŞifreBuraya';
\q
```

## 4. Node.js ve n8n Kurulumu
```
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g n8n
```

## 5. n8n için Ayrı Kullanıcı Oluştur (Güvenlik için root kullanma)
```
sudo useradd -m -s /bin/bash n8nuser
sudo passwd n8nuser
sudo mkdir /home/n8nuser/.n8n
sudo chown -R n8nuser:n8nuser /home/n8nuser/.n8n
```
## 6. n8n Systemd Servis Dosyası Oluşturma

```
nano /etc/systemd/system/n8n.service
```
```
[Unit]
Description=n8n Workflow Automation
After=network.target postgresql.service

[Service]
Type=simple
User=n8nuser
Environment=DB_TYPE=postgresdb
Environment=DB_POSTGRESDB_HOST=localhost
Environment=DB_POSTGRESDB_PORT=5432
Environment=DB_POSTGRESDB_DATABASE=n8ndb
Environment=DB_POSTGRESDB_USER=n8nuser
Environment=DB_POSTGRESDB_PASSWORD=GüçlüBirŞifreBuraya
Environment=N8N_BASIC_AUTH_ACTIVE=true
Environment=N8N_BASIC_AUTH_USER=admin
Environment=N8N_BASIC_AUTH_PASSWORD=GüçlüBirŞifreBuraya
Environment=WEBHOOK_TUNNEL_URL=https://n8n.medicalisg.com
ExecStart=/usr/bin/n8n
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```
Aktif et ve başlat:
```
sudo systemctl daemon-reload
sudo systemctl enable n8n
sudo systemctl start n8n
sudo systemctl status n8n
```
## 7. Ollama Kurulumu

```
sudo useradd -m -s /bin/bash ollama
sudo passwd ollama
```
## 9. Ollama Systemd Servis Dosyası Oluşturma

```
nano /etc/systemd/system/ollama.service
```
```
[Unit]
Description=Ollama LLM Server
After=network.target

[Service]
Type=simple
User=ollama
ExecStart=/usr/local/bin/ollama serve --host 127.0.0.1 --port 11434
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```
Aktif et ve başlat:
```
sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama
sudo systemctl status ollama
```
## 10. Ollama Modelini İndir (örnek: gpt-oss 20b)

```
sudo -u ollama ollama pull gpt-oss:20b
```
## 11. Ollama Modelini Bellekte Tutma (Opsiyonel)

```
nano /etc/systemd/system/ollama-preload.service
```
```
[Unit]
Description=Ollama Model Preload Service
After=ollama.service

[Service]
Type=simple
User=ollama
ExecStart=/usr/local/bin/ollama run gpt-oss:20b --host 127.0.0.1 --port 11434
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```
Aktif et ve başlat:
```
sudo systemctl daemon-reload
sudo systemctl enable ollama-preload
sudo systemctl start ollama-preload
```
## 12. Nginx Reverse Proxy Ayarları

```
nano /etc/nginx/sites-available/n8n.medicalisg.com
```
```
server {
    listen 80;
    server_name n8n.medicalisg.com;

    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
```
nano /etc/nginx/sites-available/ollama.medicalisg.com
```
```
server {
    listen 80;
    server_name ollama.medicalisg.com;

    location / {
        proxy_pass http://127.0.0.1:11434;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Aktif et
```
sudo ln -s /etc/nginx/sites-available/n8n.medicalisg.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/ollama.medicalisg.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```
## 13. Let's Encrypt SSL Sertifikası Alma

```
sudo certbot --nginx -d n8n.medicalisg.com
sudo certbot --nginx -d ollama.medicalisg.com
```
## 14. Son Kontroller ve Loglar

```
sudo journalctl -u n8n -f
sudo journalctl -u ollama -f
sudo journalctl -u ollama-preload -f
sudo systemctl status nginx
```
