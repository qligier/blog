---
title: "How to fix metadata of Google Pixels' raw photos"
date: 2024-01-01
draft: false
tags: ['Software', 'Photo']
description: "The blog post is about Pixel DNG Fixer, an app that fixes the metadata of DNG photos taken by Google
Pixel phones."
---

My everyday phone is a Google Pixel 4a 5G. It is a great phone with good capabilities, a jack plug and a decent
battery. I use it to take photos, and I shoot in JPG plus DNG (raw) formats. However, the raw files produced by the
Google Camera app are not correctly tagged with metadata. When I import them in Lightroom, some metadata items are
either missing or plainly wrong, which is annoying.

To illustrate that, I can read and compare the metadata of a JPG and DNG photo pair with the battle-tested
[ExifTool](https://exiftool.org) command:

```
[ExifTool]      ExifTool Version Number : 12.72                          12.72
[File]          File Name               : PXL_20231227_122735727.jpg     PXL_20231227_122735727.dng
[EXIF]          Make                    : Google                         Google
[EXIF]          Camera Model Name       : Pixel 4a (5G)                  Pixel 4a (5G)
[EXIF]          Software                : HDR+ 1.0.377695977zd           HDR+ 1.0.377695977zd
[EXIF]          Shutter Speed Value     : 1/191                          1/192
[EXIF]          Sub Sec Time            : 727                            944537
[EXIF]          GPS Latitude Ref        : North                          —
[EXIF]          GPS Longitude Ref       : East                           —
[EXIF]          GPS Altitude Ref        : Above Sea Level                —
[EXIF]          GPS Time Stamp          : 12:27:22                       —
[EXIF]          GPS Img Direction Ref   : Magnetic North                 —
[EXIF]          GPS Img Direction       : 316                            —
[EXIF]          GPS Date Stamp          : 2023:12:27                     —
[Composite]     GPS Altitude            : 501.3 m Above Sea Level        —
[Composite]     GPS Date/Time           : 2023:12:27 12:27:22Z           —
[Composite]     GPS Latitude            : 46 deg 32' 11.45" N            —
[Composite]     GPS Longitude           : 6 deg 36' 19.87" E             —
[Composite]     Hyperfocal Distance     : 2.28 m                         2.32 m
```

For that photo, we can see wrong values for the shutter speed, the GPS coordinates and the hyperfocal distance,
while
all GPS metadata items are missing in the DNG file.

Tickets have been opened on the Google issue tracker for different Pixel generations<sup><a href="#fn1">[1]</a>,<a
href="#fn2">[2]</a>,<a
href="#fn3">[3]</a>,<a href="#fn4">[4]</a></sup>, but Google does not care; tickets are closed without any
action. Since the metadata are properly saved in the JPG files, I decided to write a small application to copy
them to the DNG files.

## Pixel DNG Fixer

![The Pixel DNG Fixer application](pixeldngfixer.png "The [Pixel DNG Fixer](https://github.com/qligier/PixelDngFixer) application")

The use of _Pixel DNG Fixer_ is straightforward:
you copy all JPG and DNG file pairs into a folder, and you run the application on that folder.
JPG files should be found in the `sdcard/DCIM/Camera` phone directory, while DNG files should be found in
`sdcard/Pictures/Raw`.
Options allow you to make a backup of the DNG files before modifying them, and to delete the JPG files after processing.

The application will first match the JPG and DNG file pairs by their filename, ignoring some suffixes like `MP` that
JPG files have, but not DNG files. Then, it will copy the known broken metadata from the JPG  files to the DNG files.

The application is free and open-source; you can find it at
[github.com/qligier/PixelDngFixer](https://github.com/qligier/PixelDngFixer).
It is MIT-licensed. You are welcome to open issues if you encounter a problem
with the app, or if you want to propose a new feature.

<ol class="footnotes">
    <li id="fn1">{{< a/ext "https://support.google.com/pixelphone/thread/112616241"
        "Pixel Phone Help: Camera is no longer recording GPS data in DNG files" >}}
    </li>
    <li id="fn2">{{< a/ext "https://support.google.com/pixelphone/thread/43493374"
        "Pixel Phone Help: Pixel 4 XL not saving GPS location in photo file metadata (EXIF tags)" >}}
    </li>
    <li id="fn3">
        {{< a/ext "https://support.google.com/pixelphone/thread/156039163"
        "Pixel Phone Help: GPS-Daten fehlen im DNG-Format" >}}
    </li>
    <li id="fn4" markdown>{{< a/ext "https://support.google.com/pixelphone/thread/1314238"
        "Pixel Phone Help: added GPS coordinates to the image file meta tags" >}}
    </li>
</ol>
