---
title: "Waking up a NAS HDD with Jellyfin"
date: 2026-05-27
tags: ['Software']
description: "The blog post describes."
draft: true
---

I have at home a {{< abbr NAS >}} that I use as backup for important documents and as primary storage for my medias.
With the latter, I like to use Jellyfin, which provides a great interface!

I recently changed my old Synology model for the recent UGREEN DXP4800 Plus, on which I can now install Docker and run
Jellyfin.
That's a great improvement!
Prior to that, Jellyfin was running on an Intel {{< abbr NUC >}}, which required a bit more maintenance (especially
since it was running Windows).

I am happy with this move, but one thing has been bothering me: the hard drive of the {{< abbr NAS >}} are really slow to wake up.
From the moment you start a media in Jellyfin, it takes about 15-20 seconds to start playing, which I find unpleasant.
It was much faster with my previous setup, and I'm unsure why.

All Docker containers and their data are stored on a {{< abbr SSD >}}, while the bulk storage is on {{< abbr HDD >}}s.
So when I browse the media library, the metadata is loaded from the {{< abbr SSD >}}, but when I start a media, it has
to be read on the {{< abbr HDD >}}s.
And those are often asleep (to preserve them and save energy).

I have searched how to wake up the {{< abbr HDD >}}s when opening Jellyfin to prevent that delay, but haven't found a simple solution.
I have nonetheless found a workaround that is fast to implement and works well: I created a PHP script that
[stat](https://en.wikipedia.org/wiki/Stat_(system_call))s a directory on the {{< abbr HDD >}}s (waking them up), and configured the
webhook plugin of Jellyfin to call this script on some events (like session start).

My Docker compose file for Jellyfin initially looked like this:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:10.11.9
    container_name: jellyfin
    tty: true
    restart: unless-stopped
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /volume1/docker/jellyfin/config:/config:rw
      - /volume1/docker/jellyfin/cache:/cache:rw
      - /volume1/docker/jellyfin/logs:/logs:rw
      - /volume2/Medias:/data:ro
    ports:
     - "8096:8096"
```

We'll add a second service for the web server running the PHP script, and link them on a private network:

```yaml
services:
  jellyfin:
    # Previous configuration omitted for brevity
    networks:
      - public
      - private

  webhook:
    image: erseco/alpine-php-webserver:3.23.4
    user: 65534:65534
    container_name: webhook-server
    tty: true
    restart: always
    volumes:
      - /volume1/docker/webhook-server:/var/www/html:rw
      - /volume2/Medias:/var/www/trigger:ro
    networks:
     - private

networks:
  private:
    internal: true
  public:
```

The network 'public' is required to allow Jellyfin to be accessed from the local network, while the 'private' network
is only used between the two containers.
This makes the webhook server inaccessible from the local network, which is a good security measure (I have no reason
to expose it).
The script itself is stored on the {{< abbr SSD >}} ('volume1'), which is mounted where the container expects it ('/var/www/html').
The other volume is giving read access to the media storage on the {{< abbr HDD >}}s (that's the directory we'll stat).

The PHP script (saved on '/volume1/docker/webhook-server/index.php') is quite simple:

```php
<?php
declare(strict_types=1);
$directory = '/var/www/trigger';

http_response_code(200);

if (!is_dir($directory)) {
	http_response_code(500);
	error_log("Directory '$directory' does not exist!");
	exit;
}
clearstatcache(true, $directory);
stat($directory);
```

It simply stats the directory on the {{< abbr HDD >}}s, and clears the stat cache (that's a PHP thing) to make sure it
hits the filesystem.
We don't need more (although you could parse the notification payload and implement more complex logic).

Finally, on Jellyfin, I installed the webhook plugin (which is official).
I selected the events 'Session start' and 'Authentication successful' to trigger the webhook.

If you want to go further, you can provide a payload by either filling the 'template' textarea (), or enable the '' checkbox.
