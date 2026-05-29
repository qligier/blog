---
title: "Waking up a NAS HDD with Jellyfin"
date: 2026-05-29
tags: ['Tutorial', 'Software']
description: "The blog post describes how to trigger HDD wakeup on a NAS when opening Jellyfin."
draft: false
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
    # Previous configuration omitted for brevity. We only add this 'networks' section to the 'jellyfin' service.
    networks:
      - public
      - private

  # We create a new service to run an HTTP server with PHP support.
  webhook:
    image: erseco/alpine-php-webserver:3.23.4
    user: 65534:65534 # Run as 'nobody' user
    container_name: webhook-server
    tty: true
    restart: always
    volumes:
      - /volume1/docker/webhook-server:/var/www/html:ro
      - /volume2/Medias:/var/www/trigger:ro
    networks:
     - private

# We define the two networks used by the services.
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

It simply stats the directory on the {{< abbr HDD >}}s, and clears the stat cache to make sure it hits the filesystem.
We don't need more (although you could parse the notification payload and implement more complex logic).

Finally, on Jellyfin, I installed the webhook plugin (which is official): in _Administration_ → _Dashboard_ → 
_Plugins_, select the _Available_ ones, select _Webhook_ and click _Install_.
You need to restart the server after that (on the home _Dashboard_ page, click the _Restart_ button).
Go back to the _Plugins_ page, select the _Installed_ ones, select _Webhook_ again and click on _Settings_.
In the plugin configuration, I added a new webhook with the following settings:

- __Webhook name:__ whatever display name you want
- __Webhook Url:__ http://webhook:8080
- __Status:__ Enabled
- __Notification Type:__ _Authentication Success_ and _Session Start_

Then click on _Save_ at the bottom of the page.
That's it!
Now, when you open a Jellyfin session (e.g. by opening the web interface or any application), the webhook will be 
triggered, the PHP script triggered by that HTTP request will stat the directory on the {{< abbr HDD >}}s, and by 
the time you choose a media to play, the drives should be running.
