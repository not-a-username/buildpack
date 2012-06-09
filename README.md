# Heroku buildpack: Clojure

This is a Heroku buildpack for Clojure apps. It uses
[Leiningen](http://leiningen.org).

Note that you don't have to do anything special to use this buildpack
with Clojure apps on Heroku; it will be used by default for all
projects containing a project.clj file, though it may be an older
revision than current master. This repository is made available so
users can fork for their own needs and contribute patches back.

This particular branch enables the use of the Leiningen 2 preview.

## Usage

Example usage for an app already stored in git:

    $ tree
    |-- Procfile
    |-- project.clj
    |-- README
    `-- src
        `-- sample
            `-- core.clj

    $ heroku create --stack cedar --buildpack "http://github.com/heroku/heroku-buildpack-clojure.git#lein-2"

    $ git push heroku master
    ...
    -----> Heroku receiving push
    -----> Fetching custom buildpack
    -----> Leiningen 2 (Clojure) app detected
    -----> Installing Leiningen
           Downloading: leiningen-2.0.0-preview3
           Writing: lein script
    -----> Building with lein compile :all
           Retrieving org/clojure/clojure/1.3.0/clojure-1.3.0.pom 
               from http://repo1.maven.org/maven2/
           Retrieving org/sonatype/oss/oss-parent/5/oss-parent-5.pom 
               from http://repo1.maven.org/maven2/
           Retrieving ring/ring-jetty-adapter/1.0.0/ring-jetty-adapter-1.0.0.pom (2k)
               from http://clojars.org/repo/
           Retrieving ring/ring-core/1.0.0/ring-core-1.0.0.pom (2k)
               from http://clojars.org/repo/
    -----> Discovering process types
           Procfile declares types -> web
    -----> Compiled slug size is 10.0MB
    -----> Launching... done, v4
           http://gentle-water-8841.herokuapp.com deployed to Heroku

The buildpack will detect your app as Clojure if it has a
`project.clj` file in the root. If you use the
[clojure-maven-plugin](https://github.com/talios/clojure-maven-plugin),
[the standard Java buildpack](http://github.com/heroku/heroku-buildpack-java)
should work instead.

## Process Types

When
[declaring process types in your Procfile](https://devcenter.heroku.com/articles/procfile),
it's recommended that you specify a profile with the `with-profile`
task in order to avoid having development dependencies or tests on the
classpath. There is an empty `:production` profile provided, but you
can specify `:profiles {:production {:key "value"}}` in your
`project.clj` file if you have further configuration applicable only
in production. However, it's more common to make production-specific
values the default and simply override them in the `:dev` profile.

It's also helpful to reduce memory consumption by using the
`trampoline` task. This will cause Leiningen to calculate the
classpath and code to run for your project, then exit and execute your
project's JVM, avoiding runtime overhead:

    web: lein with-profile production trampoline run -m myapp.web

## Configuration

Apps will be built with `lein with-profile production compile :all` by
default. To specify a different build, (for example, precompiling
assets or avoiding AOT compilation or run shell commands after a
Leiningen task) check an executable `bin/build` script into your
repository and it will be run instead.

If you have dependencies which are not available in any public
repositories, try using the
[s3-wagon-private](https://github.com/technomancy/s3-wagon-private)
plugin to store them on S3.

The build process will not have access to your app's config
environment variables unless you enable
[user_env_compile](http://devcenter.heroku.com/articles/labs-user-env-compile)
using `heroku labs:enable user_env_compile`. This is required for
credentials if you use a private repository for dependencies.

## Hacking

To change this buildpack, fork it on GitHub. Push up changes to your
fork, then create a test app with `--buildpack YOUR_GITHUB_URL` and
push to it. If you already have an existing app you may use
`heroku config:add BUILDPACK_URL=YOUR_GITHUB_URL` instead.

For example, you could adapt it to generate an uberjar at build time.

Open `bin/compile` in your editor, and replace the block labeled
"fetch deps with lein" with something like this:

    echo "-----> Generating uberjar with Leiningen:"
    echo "       Running: LEIN_NO_DEV=y lein uberjar"
    cd $BUILD_DIR
    PATH=.lein/bin:/usr/local/bin:/usr/bin:/bin JAVA_OPTS="-Xmx500m -Duser.home=$BUILD_DIR" lein with-profile production uberjar 2>&1 | sed -u 's/^/       /'
    if [ "${PIPESTATUS[*]}" != "0 0" ]; then
      echo " !     Failed to create uberjar with Leiningen"
      exit 1
    fi

Commit and push the changes to your buildpack to your GitHub fork,
then push your sample app to Heroku to test. You should see:

    -----> Generating uberjar with Leiningen:

## Troubleshooting

To see what the buildpack has produced, do `heroku run bash` and you
will be logged into an environment with your compiled app available.
From there you can explore the filesystem and run `lein` commands.

You can also use [Mason](https://github.com/ddollar/mason) to test the
buildpack locally without having to push with git.
