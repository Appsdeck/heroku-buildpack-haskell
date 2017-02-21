<img src="img/haskell.png" alt="Styleguide Rails Logo" align="left" />
<br />
<h2 align="center">Deploy Haskell apps</h2>

<br />

This buildpack supports frameworks like Yesod, Snap, and Happstack with
the latest stable GHC binaries. Putting Haskell web applications online
should be easy, and now it is. Try it for yourself.

<hr />
**Note**, this buildpack does a basic build from scratch. For a
faster build, try
[mietek/haskell-on-heroku](https://github.com/mietek/haskell-on-heroku)
which detects your project dependencies and fetches a custom pre-built
sandbox.
<hr />

### Example: deploying a Snap app

Here's how to go from zero to "hello world" on Heroku. You'll need to
install the [Haskell Platform](http://www.haskell.org/platform/) and
the [Scalingo CLI](http://cli.scalingo.com/) on your local
machine, then do this:

```sh
# Generate a barebones snap app called snapdemo

mkdir snapdemo && cd $_
cabal sandbox init
cabal install snap
cabal exec snap init barebones

# Tell how to start the server

echo 'web: cabal run -- -p $PORT' > Procfile

# Create a git repo and deploy!

git init .
echo "dist\n.cabal-sandbox\ncabal.sandbox.config" > .gitignore
git add *
git commit -m 'Initial commit'

scalingo create my-haskell-app
git push scalingo master
```

**The first deploy is slowest** as the environment downloads and
bootstraps. Subsequent deploys use cached binaries and cached cabal
packages to go faster.

### Beating the Fifteen-Minute Build Limit

The first time you try to deploy a big framework like Yesod the
compilation can take so long that Heroku cuts it off. If this happens
fear not, you can build your app with an Anvil server.

### Locking Package Versions

Cabal sometimes gets confused and tries installing outdated
packages. If you have your app working locally you can constrain the
remote package versions to match your local environment. Just do this:

```sh
cabal freeze
git add cabal.config

# commit and push to fix remote build
```

### Configuring the Build

You can change build settings through environment variables.

```sh
# set the variable of your choice
scalingo env-set VARIABLE=value
```

Here are the options
<table>
<thead>
<tr><th>name</th><th>description</th><th>default</th></tr>
</thead>
<tbody>
<tr>
  <td>CLEAR_CACHE</td>
  <td>Force everything to reinstall from scratch by setting to 1.</td>
  <td>0</td>
</tr>
<tr>
  <td>EXTERNAL_CACHE</td>
  <td>Url of replacement buildpack cache tarball. Useful for Anvil.</td>
  <td></td>
</tr>
<tr>
  <td>GHC_VER</td>
  <td>GHC version to download or build</td>
  <td>7.10.3</td>
</tr>
<tr>
  <td>CABAL_VER</td>
  <td>Version of cabal-install</td>
  <td>1.22.8.0</td>
</tr>
<tr>
  <td>PREBUILT</td>
  <td>Base url for the prebuilt binary cache</td>
  <td>https://s3.amazonaws.com/heroku-ghc</td>
</tr>
<tr>
  <td>PRE_SCRIPT</td>
  <td>Path to an executable, relative to checkout root. If present, this will be chmodded (+x) and ran before anything else.</td>
  <td></td>
</tr>
<tr>
  <td>POST_SCRIPT</td>
  <td>Path to an executable, relative to checkout root. If present, this will be chmodded (+x) and ran after anything else.</td>
  <td></td>
</tr>
</tbody>
</table>


### Interacting with a running app

```sh
scalingo run bash         # shell access
scalingo run cabal repl   # Haskell repl with all app modules loaded
```

### Benefits of this buildpack

* **Latest binaries: GHC 7.10.3, cabal-install 1.22.8.0**
* Uses cabal >=1.20 features to run the app and repl
* Exposes Haskell platform binaries to your app and scripts
* Uses prebuilt binaries for speed but...
* ...can fall back to building the standard GHC distribution

### Contributing

There are a number of ways to improve this buildpack. Please see the
Github issues for ideas.

In order to contribute to the build script it will help to understand
how Scalingo's deployment process works, and how that affects GHC. Scalingo
provides three areas for storing files during build: a cache directory,
a working directory, and a build directory.

The cache, called `$CACHE_DIR` in the script, persists between
deployments. We use it to avoid building binaries more than once. The
working directory, called `$WORKING_HOME`, does not persist between
builds, or even after the build script is done. Seems we should avoid
this area, right? Well GHC has some idiosyncracies that make this area
quite useful as you will see. Finally the build destination directory,
`$BUILD_DIR`, holds the git repo the user pushes and gets copied into
what will be `/app` in the deployed application. During build it lives
in a weird nonce filename.

We want to use GHC binaries at build time to compile the app, and we
would also like those binaries to be available in the app environment
after deployment (so people can use `runhaskell` or `cabal repl`). Seems
like we should install directly to `$BUILD_DIR`. There's one problem:
GHC breaks if you move it to a new path after installation because many
of its binaries are just scripts with hard-coded full paths in them to
other GHC files. And as you remember, Heroku is going to move things in
`$BUILD_DIR` to `/app`. So the trick will be to install to `/app` in the
working directory, use GHC there, then copy that installation to the
build directory which will be renamed in the deployed application and
not notice it has been moved.

GHC is also sensitive to having `libgmp` named just right. We don't
have privileges to adjust `/usr/lib` in the deployed app so we create a
symbolic link in a place we are permitted and set linker variables in
the shell so that everything can build.

### External dependencies

Some packages have external dependencies (i.e. non-Haskell dependencies which cannot be satisfied by `cabal`). If you come across such a package, check in `contribs` to see if someone has already created patches for the dependencies. If they have, you should be able to
```sh
patch -p0 < contribs/DEP.patch
```
to patch the buildpack and proceed. If not, you'll need to modify `bin/compile` yourself. Contributing these changes back as patches is appreciated.

### Thanks

Thanks to Brian McKenna and others for their work on
[heroku-buildpack-haskell](https://github.com/puffnfresh/heroku-buildpack-haskell)
which inspired and informed this buildpack. For a history of that project's
contributions and ideas see [this article]
(http://blog.begriffs.com/2013/08/haskell-on-heroku-omg-lets-get-this.html).
