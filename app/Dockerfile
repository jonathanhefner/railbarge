# syntax = docker/dockerfile:1

ARG RUBY_VERSION

FROM ruby:$RUBY_VERSION-slim

# Since Rails 7.1, newly generated apps have the following behaviors enabled by
# default.  (See rails/rails@2b1fa89e442ecb and rails/rails@e8f481b924c799.)
# For convenience, enable them for previously generated apps as well.  If
# necessary, users can disable these behaviors by setting the values to empty
# strings in their child image.
ENV RAILS_LOG_TO_STDOUT="1" \
    RAILS_SERVE_STATIC_FILES="1"

ENTRYPOINT ["dockhand", "rails-entrypoint"]
CMD ["bin/rails", "server"]
EXPOSE 3000

RUN gem update --system --no-document --silent \
 && gem install dockhand --no-document


ONBUILD ARG RAILS_ENV
ONBUILD ENV RAILS_ENV="${RAILS_ENV:-production}"

ONBUILD ARG RAILS_ENVIRONMENTS
ONBUILD ENV BUNDLE_ONLY="default:${RAILS_ENVIRONMENTS:-$RAILS_ENV}"

ONBUILD ARG NODE_ENV
ONBUILD ENV NODE_ENV="${NODE_ENV:-production}"


ONBUILD ARG RUNTIME_PACKAGES
ONBUILD RUN --mount=type=bind,source=Gemfile,target=Gemfile \
            --mount=type=bind,source=Gemfile.lock,target=Gemfile.lock \
            --mount=type=cache,target=/var/cache/apt,sharing=locked \
            --mount=type=cache,target=/var/lib/apt,sharing=locked \
              dockhand install-packages $RUNTIME_PACKAGES --gem-runtime


ONBUILD ARG USER
ONBUILD RUN useradd "${USER:-rails}"
ONBUILD USER 1000:1000

ONBUILD ARG APP_DIR
ONBUILD WORKDIR "${APP_DIR:-/rails}"
