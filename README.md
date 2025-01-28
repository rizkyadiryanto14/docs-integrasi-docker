# Laravel 11 + Docker + Multi-Domain Deployment

## 📌 Daftar Isi
1. [Pendahuluan](#pendahuluan)
2. [Struktur Folder](#struktur-folder)
3. [Menyiapkan Docker Compose](#menyiapkan-docker-compose)
4. [Membuat Dockerfile](#membuat-dockerfile)
5. [Konfigurasi Nginx](#konfigurasi-nginx)
6. [Mengupdate `.env` di Laravel](#mengupdate-env-di-laravel)
7. [Menjalankan Docker](#menjalankan-docker)
8. [Menambahkan SSL dengan Let's Encrypt](#menambahkan-ssl-dengan-lets-encrypt)
9. [Kesimpulan](#kesimpulan)

---

## 1️⃣ Pendahuluan

Dokumentasi ini menjelaskan cara mengintegrasikan Laravel 11 dengan Docker dan mengatur banyak domain dalam satu server menggunakan **Nginx sebagai reverse proxy**.

Teknologi yang digunakan:
- **Docker + Docker Compose** untuk deployment
- **Nginx** sebagai webserver
- **PHP-FPM** untuk menjalankan Laravel
- **MySQL** sebagai database
- **Certbot (Let's Encrypt)** untuk SSL gratis

---

## 2️⃣ Struktur Folder

```
📂 webserver-docker/
│── nginx/
│   ├── domain1.conf      # Konfigurasi Domain 1
│   ├── domain2.conf      # Konfigurasi Domain 2
│
│── app1/                 # Laravel untuk domain1.com
│── app2/                 # Laravel untuk domain2.com
│
│── certbot/              # Folder SSL Let's Encrypt
│
│── docker-compose.yml     # Docker Compose Config
│── Dockerfile             # File konfigurasi Laravel
```

---

## 3️⃣ Menyiapkan Docker Compose

Buat file `docker-compose.yml`:

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:latest
    container_name: nginx_webserver
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx:/etc/nginx/conf.d
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt
      - ./app1:/var/www/domain1
      - ./app2:/var/www/domain2
    depends_on:
      - app1
      - app2

  app1:
    build:
      context: ./app1
      dockerfile: Dockerfile
    container_name: app1_laravel
    volumes:
      - ./app1:/var/www/domain1
    depends_on:
      - db

  app2:
    build:
      context: ./app2
      dockerfile: Dockerfile
    container_name: app2_laravel
    volumes:
      - ./app2:/var/www/domain2
    depends_on:
      - db

  db:
    image: mysql:8.0
    container_name: mysql_db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: laravel
      MYSQL_PASSWORD: laravel
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

---

## 4️⃣ Membuat Dockerfile

Buka `Dockerfile`:

```Dockerfile
FROM php:8.2-fpm
RUN apt-get update && apt-get install -y \
    git unzip curl libpng-dev libonig-dev libxml2-dev \
    && docker-php-ext-install pdo pdo_mysql mbstring exif pcntl bcmath gd

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www
COPY . .
RUN composer install --no-dev --optimize-autoloader
RUN chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache
CMD ["php-fpm"]
```

---

## 5️⃣ Konfigurasi Nginx

Buat `nginx/domain1.conf`:

```nginx
server {
    listen 80;
    server_name domain1.com www.domain1.com;
    root /var/www/domain1/public;
    index index.php index.html;
    location / { try_files $uri $uri/ /index.php?$query_string; }
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass app1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

Buat `nginx/domain2.conf`:

```nginx
server {
    listen 80;
    server_name domain2.com www.domain2.com;
    root /var/www/domain2/public;
    index index.php index.html;
    location / { try_files $uri $uri/ /index.php?$query_string; }
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass app2:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

---

## 6️⃣ Mengupdate `.env` di Laravel

```env
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=root
```

---

## 7️⃣ Menjalankan Docker

```bash
docker-compose up -d --build
docker ps
```

---

## 8️⃣ Menambahkan SSL dengan Let's Encrypt

```bash
docker run --rm -it \
-v ~/webserver-docker/certbot/conf:/etc/letsencrypt \
-v ~/webserver-docker/certbot/logs:/var/log/letsencrypt \
-v ~/webserver-docker/certbot/www:/var/www/certbot \
certbot/certbot certonly --webroot --webroot-path=/var/www/certbot \
-d domain1.com -d www.domain1.com
```

Lakukan langkah yang sama untuk `domain2.com`. Setelah itu, update `nginx/domain1.conf` dan `nginx/domain2.conf` untuk menggunakan **SSL**.

---

## 9️⃣ Kesimpulan

### ✔ **Docker Compose digunakan untuk Laravel, MySQL, dan Nginx**
### ✔ **Multi-domain dikonfigurasi dalam satu server**
### ✔ **SSL dari Let's Encrypt untuk keamanan HTTPS**
### ✔ **Deployment mudah dengan `docker-compose up -d`**

🚀 **Sekarang Laravel 11 siap untuk deployment multi-domain dengan Docker!**

