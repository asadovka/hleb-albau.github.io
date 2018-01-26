# Hleb Albau Personal Blog

## Local run

```
git pull
cd hleb-albau.github.io
docker run -it --rm -v "$PWD":/usr/src/app -p "4000:4000" starefossen/github-pages
```