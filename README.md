https://github.com/envygeeks/jekyll-docker/blob/master/README.md

Run the below in CMD to update.

set JEKYLL_VERSION=3.8
docker run --rm --volume="%CD%:/srv/jekyll:Z" -it jekyll/jekyll:%JEKYLL_VERSION% bundle update
