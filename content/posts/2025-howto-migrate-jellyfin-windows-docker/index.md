---
title: "How to migrate Jellyfin from Windows to Docker"
date: 2024-12-07
draft: true
tags: ['Development', 'IPF', 'Camel', 'Java', 'Swiss EPR']
description: "The blog post is describing ."
---

This blog post shows migrating from Windows to Docker, but it should be applicable to any platform.

On my new NAS, I created a folder `/volume1/docker/jellyfin` to store Jellyfin's internal data, and created three
subfolders: `config`, `cache` and `logs`.

The Docker compose file is as follows:
```yml
services:
  app:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    tty: true
    restart: always
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /volume1/docker/jellyfin/config:/config:rw
      - /volume1/docker/jellyfin/cache:/cache:rw
      - /volume1/docker/jellyfin/logs:/logs:rw
      - /volume2/Medias/Animes:/data/Animes:ro
      - /volume2/Medias/Movies:/data/Movies:ro
      - /volume2/Medias/TV:/data/TV:ro
    network_mode: bridge
    ports:
     - "8096:8096"
    environment:
     - TZ=Europe/Zurich
```

It shows the 

On Windows, the base


UPDATE TypedBaseItems SET
Path = replace(Path, '\', '/')
WHERE
Path IS NOT NULL

UPDATE TypedBaseItems SET
Path = replace(Path, 'M:\Medias\', '/data/')
WHERE
Path LIKE 'M:\Medias\%';


C:\ProgramData\Jellyfin\Server

\config\user -> /config/user
\metadata -> /config/


UPDATE TypedBaseItems SET
data = replace(data, '\\', '/')
WHERE
data IS NOT NULL;

UPDATE TypedBaseItems SET
data = replace(data, 'M:\\Vid\u00E9os\\', '/data/')
WHERE
data LIKE '%M:\\Vid\u00E9os\\%';

UPDATE TypedBaseItems SET
data = replace(data, 'C:/ProgramData/Jellyfin/Server/root/', '/config/root/')
WHERE
data LIKE '%C:/ProgramData/Jellyfin/Server/root/%';

UPDATE TypedBaseItems SET
data = replace(data, 'C:/ProgramData/Jellyfin/Server/metadata/', '/config/metadata/')
WHERE
data LIKE '%C:/ProgramData/Jellyfin/Server/metadata/%';
