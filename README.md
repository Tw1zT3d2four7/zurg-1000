# zurg-1000

a self-hosted Real-Debrid webdav server written from scratch, alternative to rclone_rd

## NOTE: zurg-1000 has a 1000 item limit on libraries and will only be provided as a container image

## config.yml

You need a `config.yml` created before you can use zurg

```yaml
# Zurg configuration version
zurg: v1

token: YOUR_TOKEN_HERE
port: 9999
concurrent_workers: 10
check_for_changes_every_secs: 15
info_cache_time_hours: 12

# List of directory definitions and their filtering rules
directories:

  # Configuration for TV shows
  shows:
    group: media # directories on different groups have duplicates of the same torrent
    filters:
      - regex: /season[\s\.]?\d/i          # Capture torrent names with the term 'season' in any case
      - regex: /Saison[\s\.]?\d/i          # For non-English namings
      - regex: /stage[\s\.]?\d/i
      - regex: /s\d\d/i           # Capture common season notations like S01, S02, etc.
      - contains: complete
      - contains: seasons
      - id: ATUWVRF53X5DA
      - contains_strict: PM19
      - contains_strict: Detective Conan Remastered
      - contains_strict: Goblin Slayer

  # Configuration for movies
  movies:
    group: media # because movies and shows are in the same group, and shows come first before movies, all torrents that doesn't fall into shows will fall into movies
    filters:
      - regex: /.*/ # you cannot leave a directory without filters because it will not have any torrents in it

  # Configuration for Dolby Vision content
  "hd movies":
    group: another
    filters:
      - regex: /\b2160|\b4k|\buhd|\bdovi|\bdolby.?vision|\bdv|\bremux/i     # Matches abbreviations of 'dolby vision'

  "low quality":
    group: another
    filters:
      - regex: /.*/

  # Configuration for children's content
  kids:
    group: kids
    filters:
      - contains: xxx       # Ensures adult content is excluded
      - id: XFPQ5UCMUVAEG         # Specific inclusion by torrent ID
      - id: VDRPYNRPQHEXC
      - id: YELNX3XR5XJQM

```

## Running

### Standalone webdav server

```bash
docker run -v ./config.yml:/app/config.yml -v zurgdata:/app/data -p 9999:9999 ghcr.io/debridmediamanager/zurg-1000:latest
```

- Runs zurg on port 9999 on your localhost
- Make sure you have config.yml on the current directory
- It creates a `zurgdata` volume for the data files

### with rclone

You will need to create a `media` directory to make the rclone mount work.

```yaml
version: '3.8'

services:
  zurg:
    image: ghcr.io/debridmediamanager/zurg-1000:latest
    restart: unless-stopped
    ports:
      - 9999:9999
    volumes:
      - ./config.yml:/app/config.yml
      - zurgdata:/app/data

  rclone:
    image: rclone/rclone:latest
    restart: unless-stopped
    environment:
      TZ: Europe/Berlin
      PUID: 1000
      PGID: 1000
    volumes:
      - ./media:/data
      - ./rclone.conf:/config/rclone/rclone.conf
    privileged: true
    cap_add:
      - SYS_ADMIN
    devices:
      - /dev/fuse
    command: "mount zurg: /data --allow-other --allow-non-empty --uid 1000 --gid 1000 --dir-cache-time 1s --poll-interval 1s --read-only --log-level INFO"

volumes:
  zurgdata:
```

Together with this `docker-compose.yml` you will need this `rclone.conf` as well on the same directory.

```
[zurg]
type = webdav
url = http://zurg:9999
vendor = other

```
