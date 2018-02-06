# Hleb Albau Personal Blog

## Local run

Build image with gems(Update every time you add a gem).
```
git pull https://github.com/hleb-albau/hleb-albau.github.io.git
cd hleb-albau.github.io
docker build -f local-dependencies -t hleb-albau-local-dependencies .
```

Run local site build
```
docker build -f local-run -t hleb-albau-local-run . \
&& docker run -it --rm -v ${PWD}:/jekyll-build -p "4000:4000" hleb-albau-local-run
```