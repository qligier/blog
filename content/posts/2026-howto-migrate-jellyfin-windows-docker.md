---
title: "How to migrate Jellyfin from Windows to Docker"
date: 2026-10-22
draft: false
tags: ['Tutorial', 'Software']
description: "The blog post is describing how to migrate a Jellyfin installation from Windows to Docker, keeping
users, settings and watch status, by copying the data and rewriting the paths it contains."
---

For a long time, my Jellyfin server was running on Windows, on an Intel {{< abbr NUC >}}.
When I moved it to Docker on my new {{< abbr NAS >}}, I wanted to keep my Jellyfin instance as it was, with a proper 
data migration between the two hosts.

The [Jellyfin documentation](https://jellyfin.org/docs/general/administration/migrate/) isn't of much help here: it
warns that the internal databases can't easily be copied or adjusted, and points to a third-party script,
[Jellyfin-Migrator](https://github.com/MMMZZZZ/Jellyfin-Migrator), which was last updated 3 years ago and last tested 
with Jellyfin 10.10.
It turns out the migration can be done by hand in three steps: copy the data files, reorganize them a bit (the folder
structure differs between the Windows installation and the Docker image), and rewrite the paths stored in a few XML
files (and optionally in the SQLite database).

This post describes a migration from Windows to Docker (with the
[official `jellyfin/jellyfin` image](https://hub.docker.com/r/jellyfin/jellyfin), which is Debian-based), but the 
same approach should work between any platforms.
I tested it with Jellyfin 12.1; the database layout changed in Jellyfin 10.11, see the note at the end of the database
section for older versions.

## Before starting

There are a few precautions to take before starting the migration process:

- __Use the same Jellyfin version on both sides__.
  Jellyfin migrates its database when it starts after an upgrade; you don't want that to happen in the middle of a
  platform migration.
  If the Windows server is outdated, update it first, then pin the same version in Docker (e.g.
  `jellyfin/jellyfin:12.1` instead of `latest`).
- __Stop the old server__ (on Windows, the service or the tray application) before copying anything, so that the SQLite
  database is in a consistent state.
- __Work on a copy__.
  Leave the old installation untouched until the new server works; you can always go back to it.
- __Make sure to read the [first library scan](#the-first-library-scan) section.__
  Some metadata could be lost in the migration, and keeping it requires more work.

## Where the files are

On Windows, the installer puts all of Jellyfin's data in `C:\ProgramData\Jellyfin\Server` (a portable installation
uses the folder given with `--datadir` instead).
The [documentation](https://jellyfin.org/docs/general/administration/configuration/#server-paths) lists the paths where
it can be, if you've got troubles finding it. 
It contains the following folders: `cache`, `config`, `data`, `log`, `metadata`, `plugins` and `root`.

The Docker image defines its own locations through environment variables:

```env
JELLYFIN_DATA_DIR=/config
JELLYFIN_CONFIG_DIR=/config/config
JELLYFIN_CACHE_DIR=/cache
JELLYFIN_LOG_DIR=/config/log
```

Note that this is the
[default configuration](https://github.com/jellyfin/jellyfin-packaging/blob/master/docker/Dockerfile), and you're free
to redefine those variables to change it.

The resulting structure is thus almost the same: everything goes in `/config`, except the cache, which has its own 
volume.

| Windows folder | Docker folder      | Content                                                         |
|----------------|--------------------|-----------------------------------------------------------------|
| `config`       | `/config/config`   | The server settings (XML & JSON files)                          |
| `data`         | `/config/data`     | The database (`jellyfin.db`), collections, playlists, server ID |
| `metadata`     | `/config/metadata` | The downloaded images and metadata                              |
| `plugins`      | `/config/plugins`  | The installed plugins and their configuration                   |
| `root`         | `/config/root`     | The library definitions                                         |
| `log`          | `/config/log`      | Copy if you want to keep older logs; usually not so important   |
| `cache`        | `/cache`           | Not needed, it will be regenerated                              |

The `data\subtitles` folder can be skipped too: it only contains subtitles extracted from the video files, which
Jellyfin extracts again when needed.

On my {{< abbr NAS >}}, I created a folder `/volume1/docker/jellyfin` to store Jellyfin's data, with two subfolders:
`config` and `cache`.
The media files are on another volume (`/volume2/Medias`), mounted read-only (because I don't want Jellyfin to 
create, modify or delete any file there).
The Docker compose file looks like:

```yaml
services:
  app:
    image: jellyfin/jellyfin:12.1 # The same version as the Windows server
    container_name: jellyfin
    tty: true
    restart: always
    devices:
      - /dev/dri:/dev/dri # Give access to the GPU, to enable hardware accelerated transcoding
    volumes:
      - /volume1/docker/jellyfin/config:/config:rw
      - /volume1/docker/jellyfin/cache:/cache:rw
      - /volume2/Medias/Animes:/data/Animes:ro
      - /volume2/Medias/Movies:/data/Movies:ro
      - /volume2/Medias/TV:/data/TV:ro
    ports:
     - "8096:8096"
```

Don't start the container yet: on an empty `/config` folder, Jellyfin would create a brand-new installation.

## Rewriting the paths

Jellyfin stores absolute paths in many places, and they now have to point to the Docker locations.
There are two kinds of paths: the paths of the media files (`M:\Medias\` becomes `/data/`) and the paths of
Jellyfin's own files (`C:\ProgramData\Jellyfin\Server\` becomes `/config/`).
In both cases, the backslashes also have to be replaced with slashes.

The media paths only have to be rewritten in the library definitions.
In the database, the media items (the movies, series, seasons and episodes) will be re-created by Jellyfin at the
first library scan anyway (more on that [below](#the-first-library-scan)), so there's no point in updating them, 
without recomputing their IDs.

### The library definitions

Each library is a folder in `root\default`, containing an `options.xml` file and one `.mblink` file per media folder.
Both contain the paths of the media folders: the `.mblink` file contains nothing but the path, and `options.xml`
lists the paths in `<PathInfos>`:

```xml
<PathInfos>
  <MediaPathInfo>
    <Path>M:\Medias\Movies</Path>
  </MediaPathInfo>
</PathInfos>
```

Remember to keep a backup of the original configuration files, or to work on a copy.

## The other settings

A few other things are worth checking before starting the new server:

- `config\network.xml`: the networking is quite different in Docker.
  If you have customized the bind addresses, the LAN networks, the known proxies or the published server URLs, review
  them.
- `config\encoding.xml`: the hardware acceleration doesn't work the same way on Linux (VAAPI or Intel QuickSync through
  `/dev/dri`) as on Windows.
  After the migration, check the settings in _Dashboard_ → _Playback_ → _Transcoding_.
  If you had set a custom FFmpeg path or a transcoding temporary folder, remove them: the image comes with its own
  FFmpeg.
- The plugins are .NET libraries and work on both platforms, but some of them have their own database or configuration
  that could contain paths too (e.g. the _Playback Reporting_ plugin).
- `data\device.txt` contains the server ID.
  Keep it: the clients will recognize the server and won't have to be added again.
- The official image runs Jellyfin as root by default.
  If you run it as another user (with `user: uid:gid` in the compose file), make sure that user owns the copied files.

Once everything is ready, copy the content of the working folder to `/volume1/docker/jellyfin/config` on the server, 
and start the container.

## The first library scan

Once started, the new server already looks familiar: the libraries, the users, the settings, the people and the watch
status are all there.
The media items still have their Windows paths, though, so they can't be played yet.
Jellyfin doesn't start a library scan by itself: start it right away (_Dashboard_ → _Libraries_ → _Scan All Libraries_).

During that scan, Jellyfin will remove all items pointing at the old paths because they're unreachable, and scan all 
library folders, resulting in the same items being added again.
On a small library (8 movies, 596 episodes in 33 TV shows) with Jellyfin 12.1, the scan took about 9 minutes.
Here's what was kept:

- the users, the settings, the libraries and the plugins;
- the people, the studios and the genres, with their images;
- the watch status: Jellyfin re-attaches the user data (played status, play count, playback position, favorites) to
  the new items, using their provider IDs (TMDb, IMDb, etc.).

And here's what was lost:

- the metadata of the movies, series, seasons and episodes, which is fetched again from the online providers:
  manually edited metadata, locked fields and images chosen by hand are definitely gone;
- the collections and the playlists: they still exist, but they are empty;
- the "date added" of the series and the seasons, which is set to the date of the scan.
  The movies and the episodes get the creation date of their file, which is close to the original one as long as the
  files kept their timestamps.

You're probably wondering why we didn't update the item paths before starting the new server?
The reason is that, while it's possible to manually migrate all those items, it requires more effort than just 
updating the paths and can't be done in SQL only, but requires a more powerful (scripting) language.
Since I wasn't using collections nor playlists, and had barely edited metadata, I was fine with that compromise and 
didn't go further.
But if that loss isn't acceptable to you, here's an explanation of how Jellyfin assigns IDs and what you'll need to 
do on the database before starting the server.

## Preventing all metadata loss

Jellyfin identifies the items by an ID that is
[computed from their type and their path](https://github.com/jellyfin/jellyfin/blob/v12.1/Emby.Server.Implementations/Library/LibraryManager.cs#L797-L819).
The paths in Jellyfin's data folder are made relative before that computation, which is why the libraries, the people,
the studios or the genres keep the same ID.
But the media files now have a different path, so they get a different ID: Jellyfin will considerate them invalid on 
re-scan, and remove them from the database.
The library scan will also recreate the entries with a different ID, and all "customized" metadata will be lost.

To prevent that, there's a bit of work to do on this database.

### The database

The database is the SQLite file `data\jellyfin.db`.
Besides the media items, it contains items that Jellyfin will keep: the libraries, the people, the studios, the
genres, etc.
Their paths point to Jellyfin's own files, and are stored in the following columns:

- `BaseItems.Path`: the path of the item.
  Many of them use the placeholders `%MetadataPath%` and `%AppDataPath%`, which only need new separators.
- `BaseItems.Data`: a JSON document with additional properties of the item, some of which are paths (e.g. the
  locations of a library).
- `BaseItemImageInfos.Path`: the images of the items (portraits, posters, logos, etc.), stored in the `metadata`
  folder.
- `ImageInfos.Path`: the profile pictures of the users.
- `MediaStreamInfos.Path`: the paths to external subtitles; null otherwise.

The paths can be rewritten with a few `UPDATE` queries, which you can run with the
[sqlite3 command-line tool](https://sqlite.org/cli.html) (`sqlite3 jellyfin.db ".read migrate.sql"`) or with
[DB Browser for SQLite](https://sqlitebrowser.org/) (in the _Execute SQL_ tab, then click _Write Changes_).
The prefix has to be replaced first, as it is written with backslashes:

```sql {title="migrate.sql"}
-- 1. Jellyfin's files: C:\ProgramData\Jellyfin\Server\ -> /config/
UPDATE BaseItems SET Path = replace(Path, 'C:\ProgramData\Jellyfin\Server\', '/config/')
    WHERE Path LIKE 'C:\ProgramData\Jellyfin\Server\%';
UPDATE BaseItemImageInfos SET Path = replace(Path, 'C:\ProgramData\Jellyfin\Server\', '/config/')
    WHERE Path LIKE 'C:\ProgramData\Jellyfin\Server\%';
UPDATE ImageInfos SET Path = replace(Path, 'C:\ProgramData\Jellyfin\Server\', '/config/')
    WHERE Path LIKE 'C:\ProgramData\Jellyfin\Server\%';
UPDATE BaseItems SET Data = replace(Data, 'C:\\ProgramData\\Jellyfin\\Server\\', '/config/')
    WHERE Data LIKE '%C:\\ProgramData\\Jellyfin\\Server\\%';

-- 2. Media files: M:\Medias\ -> /data/
UPDATE BaseItems SET Path = replace(Path, 'M:\Medias\', '/data/') WHERE Path LIKE 'M:\Medias\%';
UPDATE MediaStreamInfos SET Path = replace(Path, 'M:\Medias\', '/data/') WHERE Path LIKE 'M:\Medias\%';
UPDATE BaseItems SET Data = replace(Data, 'M:\\Medias\\', '/data/') WHERE Data LIKE '%M:\\Medias\\%';

-- 3. Remaining separators (if going from Windows to Linux): \ -> /
UPDATE BaseItems SET Path = replace(Path, '\', '/') WHERE Path LIKE '%\%';
UPDATE BaseItemImageInfos SET Path = replace(Path, '\', '/') WHERE Path LIKE '%\%';
UPDATE ImageInfos SET Path = replace(Path, '\', '/') WHERE Path LIKE '%\%';
UPDATE BaseItems SET Data = replace(Data, '\\', '/') WHERE Data LIKE '%\\%';
```

The `Data` column needs some care, because it contains JSON: the backslashes are escaped (`\` is written `\\`), and so
are the non-ASCII characters.
If Jellyfin's folder had an accented character, e.g. `C:\Données\Jellyfin\`, it would be stored as
`C:\\Donn\u00E9es\\Jellyfin\\` in the JSON, and that's what you would have to search for.
The last query only replaces the escaped backslashes (`\\`), so the other escape sequences (like `\u00E9`) are left
untouched.

To make sure you haven't missed any path, you can run the following query between the first and the second block.
It ignores the paths in the media folders, and should return nothing:

```sql {title="Finding the remaining Windows paths"}
SELECT Path FROM BaseItems WHERE Path LIKE '_:\%'
  UNION ALL SELECT Data FROM BaseItems WHERE Data LIKE '%_:\\%'
  UNION ALL SELECT Path FROM BaseItemImageInfos WHERE Path LIKE '_:\%'
  UNION ALL SELECT Path FROM ImageInfos WHERE Path LIKE '_:\%';
```

__Jellyfin 10.10 and older__ used another database, `data\library.db`, where the items are stored in the
`TypedBaseItems` table.
The same principle applies, on its `Path` and `data` columns.
You could update the Windows server to the latest version first, which converts the database, and then follow this post,
or you can try to adapt the queries to the old database (there are probably more stories of successful migration 
with the old versions that with the recent ones, so you might find more help on the web).

Now that the paths are rewritten, you need to update the IDs of modified entries.
You need to reimplement the ID generation function in your language of choice, after a bit of reverse-engineering 
(that's not too hard, given that the original function is quite short).
You also need to create a map of old IDs to new IDs, because many references to those entries exist elsewhere in the 
database.

In my test database, the references appeared in ~23 columns of 13 tables (`BaseItems.ParentId`, `BaseItems.SeriesId`,
`AncestorIds`, `UserData`, `Chapters`, `MediaStreamInfos`, `LinkedChildren`, etc.).
They come in at least three formats: uppercase with dashes (in most columns), lowercase without dashes (e.g. in
`BaseItems.PresentationUniqueKey`), and lowercase with dashes (in `UserData.CustomDataKey`).

The images of an item are stored in a folder named after its ID (e.g.
`metadata\library\97\9722df24e1dbcbe8d4d2c3c00a0512b5`): these folders have to be renamed, and the paths in
`BaseItemImageInfos.Path` updated accordingly.

I guess plugins that have their own database may also store item IDs, although I'm not aware of anyone doing it.
If you have installed such a plugin, inspect the database and configuration folders to find any potential reference 
to update as well.

Those last steps are the most tedious, since they require a lot of manual inspection to discover all reference patterns.
Take your time, use an SQL client with which you're comfortable, and go over all tables, running search queries when 
there's too many rows to have a look at all of them.
This is a trial-and-error game that can easily be reset: work on your copy of the database, stop the server, replace 
all the server's folders with your copy, start the server and verify the result in the web portal (or your preferred 
Jellyfin client).

I would advise running the library scan after such modification, otherwise there may be hidden inconsistencies that 
could bite you (much) later.
Verifying Jellyfin's logs is also a good way to ensure it encounters no errors working with the new database.

## Summary

To sum up, the migration consists of:

1. Updating the Windows server so that both sides run the same version, and stopping it.
2. Copying the `config`, `data`, `metadata`, `plugins` and `root` folders to `/config` in the container.
3. Rewriting the media paths in the `options.xml` and `.mblink` files.
4. Optional: rewriting item paths and IDs in the database, fixing all references to those IDs, and updating 
   the image paths.
5. Reviewing the network and transcoding settings.
6. Starting the container and a library scan, which re-creates the media items (unless you recomputed their IDs).
