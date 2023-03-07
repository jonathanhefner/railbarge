# syntax = docker/dockerfile:1

ARG RUBY_VERSION

FROM ruby:$RUBY_VERSION-slim

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
      gem update --system --no-document --silent \
   && gem install dockhand --no-document \
   && dockhand install-packages --buildtime \
   && dockhand transmute-to-artifacts $GEM_HOME


ONBUILD ARG RAILS_ENV
ONBUILD ENV RAILS_ENV="${RAILS_ENV:-production}"

ONBUILD ARG RAILS_ENVIRONMENTS
ONBUILD ENV BUNDLE_ONLY="default:${RAILS_ENVIRONMENTS:-$RAILS_ENV}"

ONBUILD ARG NODE_ENV
ONBUILD ENV NODE_ENV="${NODE_ENV:-production}"


ONBUILD ARG APP_DIR
ONBUILD WORKDIR "/artifacts/${APP_DIR:-/rails}"
ONBUILD RUN chmod o=u .


ONBUILD COPY Gemfile Gemfile.lock .

ONBUILD ARG BUILDTIME_PACKAGES
ONBUILD RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
            --mount=type=cache,target=/var/lib/apt,sharing=locked \
              dockhand install-packages $BUILDTIME_PACKAGES --buildtime --gem-buildtime

# Bundler cache: /usr/local/bundle/cache
ONBUILD RUN --mount=type=cache,target=/usr/local/bundle/cache,sharing=locked \
              dockhand install-gems


ONBUILD ARG PACKAGE_JSON_DIR
ONBUILD WORKDIR "${PACKAGE_JSON_DIR}"

ONBUILD COPY $PACKAGE_JSON_DIR/package.json* $PACKAGE_JSON_DIR/.node-version* .
ONBUILD RUN dockhand install-node --optional

ONBUILD COPY $PACKAGE_JSON_DIR/yarn.lock* $PACKAGE_JSON_DIR/package-lock.json* $PACKAGE_JSON_DIR/pnpm-lock.yaml* .
# Yarn cache: /usr/local/share/.cache/yarn
# NPM cache: $HOME/.npm
# PNPM cache: $HOME/.local/share/pnpm/store
ONBUILD RUN --mount=type=cache,target=/usr/local/share/.cache/yarn,sharing=locked \
            --mount=type=cache,target=/root/.npm,sharing=locked \
            --mount=type=cache,target=/root/.local/share/pnpm/store,sharing=locked \
              dockhand install-node-modules --optional

ONBUILD WORKDIR "${PACKAGE_JSON_DIR:+..}"


ONBUILD COPY --chown=1000:1000 . .
ONBUILD USER 1000:1000
ONBUILD RUN --mount=type=secret,required=false,id=config/master.key,target=config/master.key \
            --mount=type=secret,required=false,id=RAILS_MASTER_KEY,target=config/master.key \
              HOME="/tmp" dockhand prepare-rails-app --clean
ONBUILD USER root
ONBUILD RUN chown root "${PACKAGE_JSON_DIR:-.}" && chmod o=u "${PACKAGE_JSON_DIR:-.}"
