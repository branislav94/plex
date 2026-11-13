# Plex — migracija sa TrueNAS Docker-a na Dokploy VM

Uputstvo kako je Plex prebačen iz Docker kontejnera na TrueNAS-u u Dokploy VM na Proxmox-u (pve1), bez gubitka biblioteka, watch history-ja i metadata.

| | Staro | Novo |
|---|---|---|
| Host | TrueNAS (`192.168.0.101`), Docker | Dokploy VM (`192.168.0.25`), Docker Compose preko Dokploy-a |
| Config (baza) | `/mnt/storage/configs/plex` (ZFS) | `/opt/plex/config` (lokalni disk VM-a) |
| Mediji | `/mnt/storage/plex` | isti dataset, mountovan preko NFS na `/mnt/storage/plex` |
| Transcode | `/mnt/storage/plex/Plex/plex_transcode` | `/opt/plex/transcode` (lokalno) |
| Storage Box | `/mnt/StorageBOX/Plex/media` | **izbačen** |

## Glavna ideja

Plex pamti putanje **unutar kontejnera** (`/config`, `/smb_share`, `/transcode`). Dok one ostanu iste, biblioteke rade bez ponovnog skeniranja. Putanje na hostu mogu da se menjaju.

- **Mediji** ostaju na TrueNAS-u, VM ih čita preko NFS-a.
- **Config** ide na lokalni disk VM-a. Plex koristi SQLite bazu, a SQLite preko NFS/SMB zna da napravi lock probleme ili da ošteti bazu.
- **Transcode** ide lokalno, jer je preko mreže sporo.

---

## 1. Priprema na starom Plex-u (TrueNAS)

1. U Plex-u, Settings → Library, isključi:
   - **Empty trash automatically after every scan**: obavezno! Ako novi Plex krene pre nego što su mountovi gotovi, vidi prazne foldere i može da obriše metadata.
   - **Scan my library automatically**: privremeno.
2. Zaustavi stari Plex kontejner (stop, **ne** brisanje).
3. Napravi ZFS snapshot dataseta `storage/configs/plex` (i `storage/plex` za svaki slučaj). To je rollback.

## 2. NFS share na TrueNAS-u

Shares → UNIX (NFS) Shares → `/mnt/storage/plex`:

| Polje | Vrednost |
|---|---|
| Maproot User | `apps` |
| Maproot Group | `apps` |
| Mapall User/Group | prazno |
| Hosts / Networks | IP adresa Dokploy VM-a (`192.168.0.25`), ili `192.168.0.0/24` |

Svi fajlovi na datasetu su u vlasništvu UID/GID **568** (`apps`). Sa Maproot `apps` novi Plex piše kao 568, isto kao stari. Sa `plex_admin` (UID 3000) bi radilo zbog `777` prava, ali bi vremenom nastala mešavina vlasnika.

Provera (TrueNAS):
```bash
ls -ln /mnt/storage/plex | head   # treba 568 568
id plex_admin                      # 3000 — zato NE plex_admin
```

> ⚠️ Share sadrži i `backup`, `private`, `Sofware & Games`, `romm`... sve to VM vidi sa pravom pisanja. Hosts drži što užim.

## 3. NFS mount na Dokploy VM-u

```bash
apt update && apt install -y nfs-common rsync
mkdir -p /mnt/storage/plex /opt/plex/config /opt/plex/transcode

showmount -e 192.168.0.101        # treba da se vidi /mnt/storage/plex
```

Dodaj u `/etc/fstab` (gotov red je u [`fstab.example`](fstab.example)):
```
192.168.0.101:/mnt/storage/plex  /mnt/storage/plex  nfs  defaults,_netdev,nofail,x-systemd.automount  0 0
```

```bash
systemctl daemon-reload
mount -a
ls /mnt/storage/plex

# test pisanja, treba da vidiš 568 568
touch /mnt/storage/plex/test_nfs && ls -ln /mnt/storage/plex/test_nfs && rm /mnt/storage/plex/test_nfs
```

## 4. Kopiranje config-a (TrueNAS → VM)

**Stari Plex mora biti zaustavljen. Novi Plex NE SME biti pokrenut dok kopiranje ne završi.**

Provera mesta:
```bash
sudo du -sh /mnt/storage/configs/plex   # TrueNAS
df -h /opt                               # VM: treba bar 2x više slobodno
```

Sa TrueNAS shell-a:
```bash
sudo rsync -aHAX --numeric-ids --info=progress2 /mnt/storage/configs/plex/ root@192.168.0.25:/opt/plex/config/
```

Kad dođe do 100%, uradi završni prolaz sa `--delete`. On pravi identičnu kopiju i briše na VM-u sve što tamo ne treba da postoji (original na TrueNAS-u se ne dira):
```bash
sudo rsync -aHAX --numeric-ids --delete --info=progress2 /mnt/storage/configs/plex/ root@192.168.0.25:/opt/plex/config/
```
Ako drugi prolaz završi za par sekundi, sve je prekopirano.

Provera na VM-u:
```bash
ls "/opt/plex/config/Library/Application Support/Plex Media Server/"
# Preferences.xml, Metadata, Plug-in Support, ...
```

Pošto se kopira i `Preferences.xml`, server zadržava isti ID i claim, pa nema ponovnog povezivanja sa nalogom.

## 5. Pokretanje u Dokploy-u

1. Napravi projekat, pa **Create Service → Compose** (Docker Compose, izvor: Raw).
2. Nalepi [`docker-compose.yml`](docker-compose.yml).
3. **Deploy**. Domen ne treba, Plex ide direktno na port 32400. (Provider može biti i GitHub: repo `branislav94/plex`, Compose Path `./docker-compose.yml`.)

Dokploy radi preko Docker-a, pa su Stop/Start u Dokploy-u isto što i `docker stop plex` / `docker start plex` na VM-u.

## 6. Provera

```bash
docker ps | grep plex
docker logs plex --tail 30          # "libusb_init failed" se ignoriše
docker exec plex ls /smb_share/Plex # Movies, TV Shows, Anime...
```

1. Otvori `http://192.168.0.25:32400/web` (Ctrl+F5).
2. Biblioteke ne smeju imati žute ikonice ⚠️.
3. Pusti neki film, pa nešto što traži transcode.
4. Ako koristiš remote access: port forward 32400 na ruteru → `192.168.0.25`, pa proveri Settings → Remote Access.
5. Tek kad sve radi, ponovo uključi auto scan (i empty trash, ako ti treba).

Stari kontejner na TrueNAS-u: ostavi zaustavljen 1–2 nedelje, sa restart policy `no`. Dva servera sa istim ID-jem prave probleme. Briši ga tek kad budeš siguran.

---

## Backup

Config više nije na ZFS-u, pa:

1. **Svake noći kopija config-a nazad na TrueNAS.** Na VM-u: `crontab -e`, pa red iz [`backup/crontab.example`](backup/crontab.example):
   ```bash
   mkdir -p /mnt/storage/plex/backup/plex-config
   ```
   ```
   30 3 * * * rsync -a --delete --exclude 'Library/Application Support/Plex Media Server/Cache' /opt/plex/config/ /mnt/storage/plex/backup/plex-config/
   ```
   Na TrueNAS-u podesi Periodic Snapshot Task za `storage/plex`. Plex sam pravi backup baze na svaka 3 dana (Settings → Scheduled Tasks).
2. **Backup cele VM** u Proxmox-u: Datacenter → Backup → Add, za Dokploy VM, npr. jednom nedeljno.

> Oba backupa su na istom pool-u. Za pravu sigurnost treba i jedna kopija van mašine (npr. Hetzner Storage Box).

---

## Problemi na koje smo naleteli

| Simptom | Uzrok | Rešenje |
|---|---|---|
| `clnt_create: RPC: Unknown host` | u komandi ostao placeholder `TRUENAS_IP` | upiši pravu IP adresu; proveri `grep TRUENAS_IP /etc/fstab` |
| `root@dokploy:~#: command not found` | kopiran i prompt | kopiraj samo komandu |
| `touch: cannot touch ... No such file or directory` | NFS još nije mountovan | korak 3 |
| `bash: rsync: command not found` + `code 12` | rsync nije instaliran na VM-u | `apt install -y rsync` na VM-u |
| `Permission denied` za root pri rsync-u | root SSH login lozinkom isključen | privremeno `PermitRootLogin yes` u `/etc/ssh/sshd_config`, pa vratiti |
| Sve biblioteke sa ⚠️ | Plex pokrenut dok rsync nije završio | stop Plex → rsync do 100% → rsync `--delete` → start |
| `declares mutually exclusive network_mode and networks` | Dokploy (GitHub izvor) dodaje `dokploy-network`, a ona se ne slaže sa `network_mode: host` | umesto host moda koristi `ports` (vidi `docker-compose.yml`) |
| `Conflict. The container name "/plex" is already in use` | ostao kontejner iz prethodnog (Raw) deploy-a | obriši stari servis u Dokploy-u ili `docker rm -f plex`; podaci su u bind mountovima i ostaju |

## Rollback

1. Zaustavi Plex u Dokploy-u.
2. Na TrueNAS-u pokreni stari kontejner (config je netaknut, a postoji i snapshot).
3. Port forward vrati na staru IP adresu, ako je menjan.
