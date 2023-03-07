# railbarge

`railbarge` is a set of Docker images that uses [`ONBUILD`][] instructions plus
[`dockhand`][] to provide a turn-key `Dockerfile` UX for Rails applications.

[`dockhand`]: https://github.com/jonathanhefner/dockhand
[`ONBUILD`]: https://docs.docker.com/engine/reference/builder/#onbuild


For example, the following `Dockerfile` will work for many Rails applications:

  ```dockerfile
  ARG RUBY_VERSION=3.2.0

  FROM railbarge/builder:ruby-$RUBY_VERSION as builder

  FROM railbarge/app:ruby-$RUBY_VERSION
  COPY --from=builder /artifacts /
  ```

It will:

* Set `RAILS_ENV` to `production`, and set `BUNDLE_ONLY` based on `RAILS_ENV` in
  order to limit which packages and gems are installed (in both the builder and
  final application stages).

* Install buildtime-only apt packages in the builder stage [based on common
  gems][buildtime packages] in your `Gemfile`, such as `libsqlite3-dev` if using
  the `sqlite3` gem or `libpq-dev` if using the `pg` gem.

  [buildtime packages]: https://github.com/jonathanhefner/dockhand/blob/v0.1.0/lib/dockhand/command.rb#L167-L171

* Install gems as artifacts in the builder stage.

* Install Node.js in the builder stage *if* your application has a
  `.node-version` file or a `package.json` file with an `engines.node` value.

* Set `NODE_ENV` to `production`.

* Install Node.js modules as artifacts in the builder stage *if* your
  application has a Yarn, NPM, or PNPM lock file.

* Copy your application code as an artifact in the builder stage.

* Precompile gem and application code with [`bootsnap`][] if present.

  [`bootsnap`]: https://rubygems.org/gems/bootsnap

* Precompile assets with a dummy `SECRET_KEY_BASE` if your application uses the
  asset pipeline.  To use the actual `SECRET_KEY_BASE` from your credentials
  file, set the `RAILS_MASTER_KEY` or `config/master.key` build secret:

    ```console
    $ RAILS_MASTER_KEY="..." docker build --secret id=RAILS_MASTER_KEY -t my_cool_app .
    $ docker build --secret id=config/master.key -t my_cool_app .
    $ docker build --secret id=config/master.key,src=config/credentials/production.key -t my_cool_app .
    ```

* Fix binstubs in your application's `bin/` directory if they were generated on
  Windows.

* Install runtime apt packages in the final application stage [based on common
  gems][runtime packages] in your `Gemfile`, such as `libsqlite3-0` if using the
  `sqlite3` gem or `postgresql-client` if using the `pg` gem.

  [runtime packages]: https://github.com/jonathanhefner/dockhand/blob/v0.1.0/lib/dockhand/command.rb#L160-L165

* Set `RAILS_LOG_TO_STDOUT` and `RAILS_SERVE_STATIC_FILES` for Rails
  applications generated prior to Rails 7.1.  (See [`rails/rails@2b1fa89`][]
  and [`rails/rails@e8f481b`][].)

  [`rails/rails@2b1fa89`]: https://github.com/rails/rails/commit/2b1fa89e442ecb99e262a34eb11bc1110912cd49
  [`rails/rails@e8f481b`]: https://github.com/rails/rails/commit/e8f481b924c799bddab0b0d153b04e014f8193f3

* Add an unprivileged user named `rails`, and set `USER` as `rails`.

* Set `WORKDIR` for the final application stage as `/rails`.

* Set `ENTRYPOINT` as your application's `bin/docker-entrypoint` and `CMD` as
  `bin/rails server`.  If your application doesn't have a `bin/docker-entrypoint`
  file, a fallback will be used that injects a call to `bin/rails db:prepare`
  whenever the given command is `bin/rails server` (or an alias thereof).

* Expose port 3000.

* Lastly, the `Dockerfile` copies the artifacts from the builder stage to the
  final application stage with `COPY --from=builder /artifacts /`.

Behind the scenes, the above steps use [`--mount=type=cache`][] where
appropriate to reduce build times and to prevent unnecessary files from being
included in the final image.

[`--mount=type=cache`]: https://docs.docker.com/engine/reference/builder/#run---mounttypecache


Many of the above steps can be configured via build args.  For example:

  ```dockerfile
  # Override the Rails environment and the gem group to install (default: "production")
  ARG RAILS_ENV="staging"

  # Specify multiple Rails environments to install gems for (default: RAILS_ENV)
  ARG RAILS_ENVIRONMENTS="staging:test"

  # Specify additional apt packages for the build stage
  ARG BUILDTIME_PACKAGES="libmagickwand-dev"

  # Specify additional apt packages for the final application stage
  ARG RUNTIME_PACKAGES="imagemagick sudo"

  # Override the Node.js environment (default: "production")
  ARG NODE_ENV="development"

  # Specify the subdirectory of your application that contains the package.json
  # file, such as for a dedicated front-end client
  ARG PACKAGE_JSON_DIR="frontend"

  # Override the user name (default: "rails")
  ARG USER="dude"

  # Override the application directory name and WORKDIR (default: "/rails")
  ARG APP_DIR="/my_cool_app"


  ARG RUBY_VERSION=3.2.0

  FROM railbarge/builder:ruby-$RUBY_VERSION as builder

  FROM railbarge/app:ruby-$RUBY_VERSION
  COPY --from=builder /artifacts /
  ```

(Note that the `ARG` statements must come **before** the first `FROM` statement;
otherwise, the `railbarge` images will not see them.)


And, of course, you can append instructions to the stages themselves.  For
example, you can...

* Disable `RAILS_LOG_TO_STDOUT` or `RAILS_SERVE_STATIC_FILES` for Rails
  applications generated prior to Rails 7.1:

    ```dockerfile
    FROM railbarge/builder:ruby-2.7.7 as builder

    FROM railbarge/app:ruby-2.7.7
    ENV RAILS_LOG_TO_STDOUT=""
    ENV RAILS_SERVE_STATIC_FILES=""
    COPY --from=builder /artifacts /
    ```

* Enable [YJIT][]:

    ```dockerfile
    FROM railbarge/builder:ruby-3.2.0 as builder

    FROM railbarge/app:ruby-3.2.0
    ENV RUBY_YJIT_ENABLE="1"
    COPY --from=builder /artifacts /
    ```

  [YJIT]: https://github.com/Shopify/yjit

* Switch to [jemalloc][]:

    ```dockerfile
    ARG RUNTIME_PACKAGES="libjemalloc2"

    FROM railbarge/builder:ruby-3.2.0 as builder

    FROM railbarge/app:ruby-3.2.0
    ENV LD_PRELOAD="/usr/lib/x86_64-linux-gnu/libjemalloc.so.2"
    ENV MALLOC_CONF="dirty_decay_ms:1000,narenas:2,background_thread:true"
    COPY --from=builder /artifacts /
    ```

  [jemalloc]: https://jemalloc.net/

* Override `ENTRYPOINT` or `CMD`, such as to start [`foreman`][]:

    ```dockerfile
    FROM railbarge/builder:ruby-3.2.0 as builder

    FROM railbarge/app:ruby-3.2.0
    COPY --from=builder /artifacts /
    ENTRYPOINT ["bin/my-entrypoint"]
    CMD ["foreman", "start"]
    ```

  [`foreman`]: https://ddollar.github.io/foreman/

* Run an extra NPM build step, such as for a dedicated front-end client:

    ```dockerfile
    ARG PACKAGE_JSON_DIR="client"

    FROM railbarge/builder:ruby-3.2.0 as builder
    RUN cd client \
     && npm run build \
     && mv build/* ../public

    FROM railbarge/app:ruby-3.2.0
    COPY --from=builder /artifacts /
    ```

* Install Node.js as a build artifact so that it will be available at runtime,
  such as to use [Puppeteer][]:

    ```dockerfile
    # Install Chromium for Puppeteer:
    ARG RUNTIME_PACKAGES="chromium"

    FROM railbarge/builder:ruby-3.2.0 as builder
    # Install Node.js as a build artifact:
    RUN dockhand install-node --prefix=/artifacts/usr/local

    FROM railbarge/app:ruby-3.2.0
    # Point Puppeteer to Chromium:
    ENV PUPPETEER_EXECUTABLE_PATH="/usr/bin/chromium"
    # Copy installed Node.js along with other artifacts:
    COPY --from=builder /artifacts /
    ```

    [Puppeteer]: https://pptr.dev/


## License

[MIT License](LICENSE.txt)
