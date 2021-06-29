FROM ruby:2.7
WORKDIR /site
ARG FLAGS=""

## Base Image
base-image:
  RUN apt-get update 
  RUN apt-get install gcc cmake imagemagick -y
  RUN gem install bundler -v "~>1.0" && gem install bundler jekyll

  # # diagrams and stuff
  RUN apt-get install cabal-install -y
  RUN cabal update
  # ToDo: This takes forever, and there is a static binary offered
  RUN cabal install pandoc-plot --dependencies-only --force-reinstalls
  RUN cabal install pandoc-plot --force-reinstalls
  RUN cp /root/.cabal/bin/* /usr/bin/

  RUN apt-get install python3-matplotlib libvips-dev python3-pip npm -y
  RUN pip3 install pandocfilters
  RUN npm install -g markdownlint-cli 

  # Vale
  RUN curl -sfL https://install.goreleaser.com/github.com/ValeLint/vale.sh | sh -s v2.10.3
  RUN cp /site/bin/vale /bin
  SAVE IMAGE --push agbell/website-base:latest #Acts as a cache

deps:
    ## moved to dockerfile for build speed
    FROM agbell/website-base:latest

## Website
website-update:
  FROM +deps
  COPY website .
  RUN rm Gemfile.lock
  RUN bundle install
  RUN bundle update
  SAVE ARTIFACT Gemfile.lock AS LOCAL website/Gemfile.lock

website-install:
    FROM +deps
    COPY website/Gemfile .
    COPY website/Gemfile.lock .
    RUN bundle install --retry 5 --jobs 20
    SAVE IMAGE --push agbell/website-install:latest #Acts as a cache

website-build:
  #FROM +website-install
  FROM agbell/website-install
  COPY website .
  RUN bundle exec jekyll build $FLAGS
  SAVE ARTIFACT _site AS LOCAL build/site

website-docker:
    FROM +website-install
    CMD JEKYLL_ENV=production bundle exec jekyll serve --incremental -H 0.0.0.0 -P 4001
    SAVE IMAGE earthly-website

website-run:
  LOCALLY
  # BUILD +website-docker
  RUN docker run -p 4001:4001 -v $(pwd)/website:/site earthly-website

## Blog
blog-update:
  FROM +deps
  COPY blog .
  RUN rm Gemfile.lock
  RUN bundle install
  RUN bundle update
  SAVE ARTIFACT Gemfile.lock AS LOCAL blog/Gemfile.lock

blog-install:
  FROM +deps
  COPY blog/Gemfile .
  COPY blog/Gemfile.lock .
  RUN bundle install --retry 5 --jobs 20
  SAVE IMAGE --push agbell/blog-install:latest #Acts as a cache

blog-lint:
  #FROM +blog-install
  FROM agbell/blog-install
  COPY .vale.ini .
  COPY blog/.markdownlintrc .
  COPY .github .github
  COPY blog blog
  RUN vale --output line --minAlertLevel error ./blog/_posts/*.md
  IF grep '[“”‘’]' ./blog/_posts/*.md
    RUN echo "Fail: Remove curly quotes and use straight quotes instead" && false
  END  
  IF grep 'imgur.com' ./blog/_posts/*.md
    RUN echo "Fail: external image link" && false
  END
  RUN markdownlint "./blog/_posts/*.md"

blog-lint-apply:
  LOCALLY
  RUN sed -i -E "s/“|”/\"/g" ./blog/_posts/*.md
  RUN sed -i -E "s/‘|’/'/g" ./blog/_posts/*.md
  # remove double spaces
  RUN sed -i -E "s/\.\s\s(\w)/. \1/g" ./blog/_posts/*.md
  RUN sed -i -E "s/\?\s\s(\w)/? \1/g" ./blog/_posts/*.md
  RUN vale --output line --minAlertLevel error ./blog/_posts/*.md
  RUN cd blog && markdownlint --fix "./_posts/*.md"


blog-writing-suggestions:
  FROM agbell/blog-install
  COPY .vale.ini .
  COPY blog/.markdownlintrc .
  COPY .github .github
  COPY blog blog
  RUN vale ./blog/_posts/*.md

blog-build:
  #FROM +blog-install
  FROM agbell/blog-install
  COPY blog .
  RUN JEKYLL_ENV=production bundle exec jekyll build $FLAGS
  SAVE ARTIFACT _site AS LOCAL build/site/blog

blog-docker:
  FROM +blog-install
  CMD bundle exec jekyll serve -H 0.0.0.0 --future --incremental -P 4002
  SAVE IMAGE earthly-blog

blog-interactive:
  FROM +blog-install
  COPY blog .
  RUN --interactive /bin/bash

blog-run:
  LOCALLY
  WITH DOCKER --load=+blog-docker
    RUN docker rm -f earthly-blog && \
      docker run -p 4002:4002 -v $(pwd)/blog:/site --rm --name earthly-blog earthly-blog
  END

blog-local:
  LOCALLY
  RUN cd blog && bundle exec jekyll serve --future --incremental --profile -H 0.0.0.0 -P 4002

## Utils

clean:
  LOCALLY
  RUN rm -rf build website/_site website/.sass-cache website/.jekyll-metadata website/.jekyll-cache
  RUN rm -rf build blog/_site blog/.sass-cache blog/.jekyll-metadata blog/.jekyll-cache

# doesn't work
shell: 
  LOCALLY    
  BUILD +docker
  RUN --interactive docker run -p 4001:4001 -v $(pwd)/website:/site -it --entrypoint=/bin/bash earthly-website

# get shell, but no volume mount
static-shell:
  FROM +jekyll-install
  COPY website .
  RUN --interactive /bin/bash

## Dev Build
dev-build:
  BUILD +blog-lint
  BUILD --build-arg FLAGS="--future" +website-build 
  BUILD --build-arg FLAGS="--future" +blog-build

# Prod Build
build:
  BUILD +blog-lint
  BUILD +website-build
  BUILD +blog-build

# Publish by pushing published site to seperate git repo
manual-publish:
  LOCALLY
  BUILD +website-build
  BUILD +blog-build
  ARG branch=main
  RUN git clone https://github.com/earthly/website-output.git || true
  RUN cd website-output \
       && git checkout -b $branch || true \
       && git rm -rf . || true \
       && git clean -fxd 
  RUN cp -a build/site/. website-output/
  RUN cd website-output && ls
  RUN cd website-output \
    && git add -A \
    && git commit -m "Latest website - manual publish" || exit 0 \ 
    && git push 
   
new-post:
  LOCALLY
  ARG name="one-two-three"
  RUN cat ./blog/_posts/2029-01-01-checklist.md > ./blog/_posts/$(date -v +7d +"%Y-%m-%d")-$name.md
  RUN sed -i -E "s/published: False//g" ./blog/_posts/$(date -v +7d +"%Y-%m-%d")-$name.md
  RUN mkdir ./blog/assets/images/$name
  RUN cp ./blog/assets/images/default-header.jpg ./blog/assets/images/$name/header.jpg

# this looks for places you can manually add links to your page using internal-links in the post frontmatter
link-opportunity:
  FROM agbell/website-base:latest
  COPY blog blog 
  ARG NAME="2020-09-10-better-builds.md"
  RUN pip3 install python-frontmatter
  RUN python3 ./blog/_util/suggest-links.py ./blog/_posts/$NAME
