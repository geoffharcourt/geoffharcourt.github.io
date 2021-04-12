---
layout: post
title: Caching dependencies for speed and maximum reuse in Buildkite
---

After tearing through half of the CI products in the market and being
dissatisfied with all of them for one reason or another, we've been very happy
using [Buildkite](https://buildkite.com) for over a year now. Buildkite has a
clever "bring your own infrastructure" approach that they make much less scary
by providing a really nice CloudFormation template that does almost everything
for you to provision an auto-scaling fleet of EC2 machines to perform jobs.

One of the hardest parts of getting CommonLit's primary application into
Buildkite for CI was adapting it to work with Docker. We deploy in production to
Heroku and prefer to leverage Heroku's platform capabilities without Docker, so
our use of Docker is CI-only. We upgrade dependencies frequently and use Depfu
to update one package (or a small number of related dependencies) at a time, so
we wanted a system where feature branch builds wouldn't be spending a lot of
time installing dependencies that had already been installed in other builds.
This is important to us because we don't want to waste money on needless CI
work, but also because it slows down our feedback cycle.

Here's how we get our Ruby gems and NPM packages via Yarn built with the least
amount of work possible:

At the very start of our build, we have a job each for Bundler and Yarn. These
jobs each run with their own custom Dockerfile whose sole job is to get
dependencies installed.

```dockerfile
# Use the previously built gems to avoid reinstalling all gems if one changes
# and the Docker layer cache is broken.

ARG RUBY_VERSION

ARG BASE_IMAGE=builderimagename:ruby-${RUBY_VERSION}-alpine3.12
ARG CACHE_IMAGE=${BASE_IMAGE}

FROM $CACHE_IMAGE AS gems

RUN mkdir -p /bundle

# Use the builder image which has the development dependncies to build native
# extentions, etc.
FROM builderimagename:ruby-$RUBY_VERSION-alpine3.12 as built

ENV BUNDLE_PATH=/bundle \
    GEM_HOME=/bundle
ENV PATH $GEM_HOME/bin:$GEM_HOME/gems/bin:$PATH

COPY --from=gems /bundle /bundle

COPY .diffend.yml Gemfile Gemfile.lock ./

RUN bundle config set --local frozen 'true'
RUN bundle install --jobs="$(getconf _NPROCESSORS_ONLN)" && bundle clean

FROM alpine:3.12

WORKDIR /app

COPY --from=built /bundle /bundle
```

Here's what this image does:

We have a base "builder" image that contains the development software needed to
install gems and their C extensions. This includes things like Postgres
development headers, Readline, tzdata, etc. If there's no prior cache, we use
the builder image as the base (this enables us to bootstrap the pipeline). If
we have a prior cache image from a prior gem build, we use that as our starting
point, and copy that image's gems over. The cache enabled us to only install the
one gem that might have changed instead of installing all of them, similar to
the speed with which you can run `bundle install` in your development
environment. We want to avoid running `bundle install` in an empty-cache
scenario, which could take several minutes as gems with heavy C extensions need
to be built, etc.

We build our gems in the `/bundle` directory. At the end of the build, we copy
the `/bundle` directory into a new, minimum Alpine Linux Docker image. We do
this so that we the smallest image possible that we can reuse later in the build
and in future builds without carrying around all the development tooling that is
only needed to get the gems installed. This copying to a new "gems and nothing
else" image saves about 250MB of deadweight that we'd otherwise be downloading
repeatedly in other containers and needing to load into memory to boot the
relevant container.

(Note that Buildkite doesn't yet support mounting volumes in build steps, but if
that happened we could leverage that to speed up the use of old builds for even
more efficiency.)

Our job for Yarn copies `/node_modules` to a clean Alpine image in a similar
way. The job to compile JS/CSS assets copies `public/assets`, `public/packs`,
and some Webpacker caching intermediaries so that containers running system
tests can avoid building assets (unfortunately very expensive in our app) but
leverage the same asset-free base image that our unit specs use.

The gems image will later get mounted in other Docker images that do Ruby
testing or linting. These images never run `bundle install` themselves, so they
get right to performing valuable work. Here's an abbreviated version of our
`docker-compose.yml` file:

```yaml
services:
  gems:
    build:
      context: .
      dockerfile: Dockerfile.gems
      args:
        - BUNDLER_VERSION
        - RUBY_VERSION
    volumes:
      - /bundle

  app:
    build:
      context: .
      args:
        - BUNDLER_VERSION
        - NODE_MAJOR
        - RUBY_VERSION
    working_dir: /app
    volumes_from:
      - gems
    depends_on:
      - gems
```

Whenever a test step needs do run a Ruby process, we mount the gems image back
at the `/bundle` mount point so that the gems are all there where the app
expects them to be. We do the same thing with Yarn in another process, only
mounting the NPM packages when they are needed (Jest specs, Typescript type
checking, precompiling assets).

Here's what the Buildkite steps look like to leverage this setup:

```yaml
steps:
  - label: ":docker: :rubygems: Bundler"
    key: docker-gem-build
    plugins:
      - commonlit/docker-compose:
          build: gems
          image-repository: ourrepo/gems
          use-prior-image: true
          cache-from:
            - gems:ourrepo/gems:ruby-$RUBY_VERSION-$GEMFILE_LOCK_HASH-alpine3.12
          args:
            - CACHE_IMAGE=ourrepo/gems:ruby-$RUBY_VERSION-trunk-alpine3.12
          push:
            - gems:ourrepo/gems:ruby-$RUBY_VERSION-$GEMFILE_LOCK_HASH-alpine3.12


  - label: ":rails: :rspec: Rails Unit and Request Specs %n"
    command: "bin/ci/run_parallel_unit_specs"
    parallelism: 10
    depends_on:
      - docker-build
      - docker-gem-build
    plugins:
      - docker-compose#v3.2.0:
          run: app
          pull:
            - app
            - gems
```

Our Docker images are always tagged with the Ruby version so that when we update
Ruby, we don't accidentally comingle old images with the wrong gems. We also
mark the gem image we build with a hash of `Gemfile.lock` so that in future
builds we can pull the exact image we need for other code when the gems haven't
changed at all. We set `GEMFILE_LOCK_HASH` (and equivalents for `yarn.lock` and
our browser assets) in a hook before the job starts so that we can use that hash
throughout the build.

We use a fork of the Buildkite Docker Compose plugin in our build steps. The
only difference our fork brings to the table is that if the `cache-from` image
exists in our repository, we skip all later steps, tag that image as the build's
"gems" image, and pass the step. This allows us to avoid booting the Docker
container and running `bundle install` at all only to make no changes. In cases
where no gems change in a build (or no NPM packages or no browser assets), the
steps that rely on these dependencies start promptly and enable faster feedback.
We see even greater gains in avoiding rebuilding browser assets for pull
requests that don't alter any JavaScript or CSS. Our image with browser assets
gets mounted to the application for system tests and we get to work.

Using this practice enables us to maximize the share of our CI time and spend
that's put into running tests instead of preparing dependencies or building
assets.
