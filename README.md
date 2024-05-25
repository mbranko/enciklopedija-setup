# Instalacija servera za Srpsku enciklopediju

Ovaj dokument opisuje instalaciju i podešavanje servera za online izdanje
Srpske enciklopedije.

## Hardverska konfiguracija

* CPU: Intel Core i7
* RAM memorija: 128 GB
* Diskovi: 4x 1TB SATA SSD
* Ethernet: na matičnoj ploči
* Rack-mount kućište

## Fajl sistem

Instaliran je Debian 12.2.0, bez grafičkog interfejsa, sa sshd serverom. Tokom 
instalacije je kreiran i korisnik `branko` sa početnim direktorijumom u 
`/home/branko`. Raspored particija po diskovima je sledeći:

* `/dev/sda`:
  - `sda1`: `/boot/efi`
  - `sda2`: `swap`
  - `sda3`: `/`
* `/dev/sdb`:
  - `sdb1`: `/opt`
* RAID1 niz: `/dev/sdc1` i `/dev/sdd1`
  - `/dev/md0`: `/var`

Svi podaci kojima će rukovati softver nalaziće se u `/var` direktorijumu (na 
RAID1 nizu). U `/opt` direktorijumu nalaze se bekapi podataka. Prvi disk 
(`sda`) sadrži sistemski softver.

## Mreža

Matična ploča ima jedan Ethernet port i on je konfigurisan sa statičkom IP 
adresom 192.168.1.25. U fajlu `/etc/network/interfaces` nalazi se sledeća
konfiguracija:
```
auto lo
iface lo inet loopback

allow-hotplug enp2s0
iface enp2s0 inet static
  address 192.168.1.25/24
  gateway 192.168.1.1
  dns-nameservers 8.8.8.8 8.8.4.4
```

Instaliran je i konfigurisan `ufw` softver za firewall.
```bash
apt-get install ufw
systemctl enable ufw
ufw allow ssh
ufw allow http
ufw allow https
```

Dopušten je ssh pristup za root korisnika uz uslov da se koristi ssh ključ
(prijava lozinkom nije dozvoljena). U fajlu `/etc/ssh/sshd_config` nalazi se
red:
```
PermitRootLogin prohibit-password
```

U `/root/.ssh/authorized_keys` nalaze se javni ključevi korisnika koji imaju
ssh pristup.

## Docker servis

Docker je instaliran prema zvaničnom uputstvu za instalaciju Dockera na Debian
(sve kao `root`):
```bash
# Dodaj Docker GPG kljuc
apt-get update
apt-get install ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Dodaj repozitorijum za apt
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Instaliraj Docker
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Proveri instalaciju
docker run hello-world

# Dodaj branka u docker grupu
usermod -aG docker branko

# Automatsko pokretanje Dockera prilikom podizanja sistema
systemctl enable docker.service
systemctl enable containerd.service
```

Dodat je konfiguracioni fajl `/etc/docker/daemon.json` za podešavanje Dockerovog loga:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

## Servisi aplikacije

Aplikacija koja se koristi za online izdanje Srpske enciklopedije je 
[BookStack](https://www.bookstackapp.com). Za pokretanje ove aplikacije koristi
se Docker slika `lscr.io/linuxserver/bookstack`. Aplikacija se sastoji iz dva
kontejnera; pored `bookstack` potreban je i `mariadb` kontejner za relacionu
bazu podataka MariaDB.

BookStack kontejner ima HTTP server koji sluša na portu 80. SSL terminacija se
obavlja na reverznom proksiju čija javna adresa je 89.216.122.103, a na ovu 
adresu upućuje i Internet domen aplikacije `srpskaenciklopedija.rs`.

Svi skriptovi i podaci aplikacije nalaze se u direktorijumu 
`/var/dockersites/bookstack`:
```
/var/dockersites/bookstack
├── backup.sh
├── config
├── data
├── docker-compose.yml
├── init
└── sr
```

Sadržaj direktorijuma je sledeći:
* `config`: konfiguracija BookStack aplikacije
* `data`: fajlovi koje koristi MariaDB
* `init`: skript za inicijalizaciju baze podataka
* `sr`: fajlovi sa prevodom korisničkog interfejsa na srpski jezik
* `docker-compose.yml`: konfiguracija Docker kontejnera
* `backup.sh`: pravljenje bekapa

Za pokretanje i zaustavljanje aplikacije koristi se `docker-compose`, sa
konfiguracijom u fajlu `docker-compose.yml`. Ovo je njegov sadržaj:

```yaml
version: "3.8"
services:
  bookstack:
    image: lscr.io/linuxserver/bookstack
    container_name: bookstack
    environment:
      - PUID=1000
      - PGID=1000
      - APP_URL=https://srpskaenciklopedija.rs
      - DB_HOST=bookstack_db
      - DB_PORT=3306
      - DB_USER=bookstack
      - DB_PASS=bookstack
      - DB_DATABASE=bookstack
      - APP_LANG=sr
      - APP_AUTO_LANG_PUBLIC=false
    volumes:
      - bookstack-config:/config
      - bookstack-serbian:/app/www/lang/sr
    restart: unless-stopped
    depends_on:
      - bookstack_db
    ports:
      - "80:80"
  
  bookstack_db:
    image: mariadb:10.10
    container_name: bookstack_db
    environment:
      - MARIADB_ROOT_PASSWORD=bookstack
      - MARIADB_DATABASE=bookstack
      - MARIADB_USER=bookstack
      - MARIADB_PASSWORD=bookstack
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    volumes:
      - bookstack-data:/var/lib/mysql
      - bookstack-init:/docker-entrypoint-initdb.d
    restart: unless-stopped

volumes:
  bookstack-data:
    name: bookstack-data
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      device: './data'
  bookstack-init:
    name: bookstack-init
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      device: './init'
  bookstack-config:
    name: bookstack-config
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      device: './config'
  bookstack-serbian:
    name: bookstack-serbian
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      device: './sr'
```

Pokretanje aplikacije se vrši komandom:
```bash
docker compose up -d
```

Zaustavljanje aplikacije se vrši komandom:
```bash
docker compose down
```

U slučaju restarta sistema Docker kontejneri će se automatski ponovo 
pokrenuti.

## Rezervna kopija podataka

Bekap se generiše pomoću skripta `/var/dockersites/bookstack/backup.sh`:
```bash
#!/bin/bash
BACKUP_DIR=/opt/backup/bookstack
NOW=$(date +"%Y-%m-%d")
[ ! -d $BACKUP_DIR ] && mkdir -p $BACKUP_DIR
docker exec bookstack_db sh -c 'exec mysqldump bookstack -u bookstack -pbookstack --no-tablespaces' | gzip > $BACKUP_DIR/bookstack-$NOW.sql.gz
find $BACKUP_DIR/bookstack* -mtime +100 | xargs rm

# TODO: another location for backups
# scp $BACKUP_DIR/bookstack-$NOW.sql.gz server1:/opt/backup/bookstack
```

Automatsko pokretanje skripta za bekap svakog dana u 06:00 je konfigurisano u 
`/etc/crontab`:
```
0 6     * * *   root    /var/dockersites/bookstack/backup.sh
```

Bekap fajlovi se čuvaju za poslednjih 100 dana u direktorijumu 
`/opt/backup/bookstack`.

**TODO:** trebalo bi, kao deo procedure za bekap, ove fajlove kopirati i na 
rezervnu lokaciju. Poslednji red u skriptu za bekap je primer kako bi se
to moglo realizovati.
