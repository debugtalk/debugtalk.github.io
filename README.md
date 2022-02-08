# My Blog

URL: https://debugtalk.com

## Workflow

### Add post

create or update posts in `content/post` directory.

### Build & Local preview

```bash
$ rm -rf public
# build
$ hugo --minify
# preview
$ hugo server -D
```

`-D, --buildDrafts`: include content marked as draft

Web Server is available at http://localhost:1313/ (bind address 127.0.0.1).

### Publish to Github Pages

git commit changed content and push to origin, the website will be built on Github Actions and deployed to github pages.

```bash
$ git push origin main
```

## Update theme

```bash
$ git submodule update --init --recursive
```
