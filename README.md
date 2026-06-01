Those are the sources of my blog.

__Serve the blog with docker:__<br>
```sh
docker run --rm -it -p 1313:1313 -v .:/project:rw ghcr.io/gohugoio/hugo:latest server --buildDrafts --buildFuture --bind 0.0.0.0 --disableLiveReload --noChmod --noTimes --cleanDestinationDir
```

File icons are from [Bootstrap Icons](https://icons.getbootstrap.com).
