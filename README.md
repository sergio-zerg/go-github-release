[![travis Status](https://travis-ci.org/mh-cbon/go-github-release.svg?branch=master)](https://travis-ci.org/mh-cbon/go-github-release)[![Appveyor Status](https://ci.appveyor.com/api/projects/status/github/mh-cbon/go-github-release?branch=master&svg=true)](https://ci.appveyor.com/project/mh-cbon/go-github-release)

# go-github-release

HOWTO implement a feature complete release software flow over github.

Aka, s/a software factory/[ma petite entreprise](https://youtu.be/tgOujR6_3Vc?t=37s)/.

Made with the power of `go`,
built with [unix philsophy](https://en.wikipedia.org/wiki/Unix_philosophy),
it provides concerns-centric lightweight tools
that can benefit to any project of any language,
[Issue #15](https://github.com/mh-cbon/go-github-release/issues/15)

<a href="https://app.codesponsor.io/link/aHBhB5M68Fescjjp9VC9TtJs/mh-cbon/go-github-release" rel="nofollow"><img src="https://app.codesponsor.io/embed/aHBhB5M68Fescjjp9VC9TtJs/mh-cbon/go-github-release.svg" style="width: 888px; height: 68px;" alt="Sponsor" /></a>

# TOC
- [TLDR;](#tldr)
- [head first](#head-first)
  - [a github account](#a-github-account)
  - [a bintray account](#a-bintray-account)
  - [windows users](#windows-users)
  - [debian users](#debian-users)
  - [rpm users](#rpm-users)
- [a dummy project](#a-dummy-project)
- [maintaining a changelog](#maintaining-a-changelog)
  - [In practice](#in-practice)
- [maintain a README](#maintain-a-readme)
  - [In practice](#in-practice-1)
- [bump a package](#bump-a-package)
  - [In practice](#in-practice-2)
  - [notes about .version.sh](#notes-about-versionsh)
- [github releases](#github-releases)
- [various other useful cross-platform tools](#various-other-useful-cross-platform-tools)
  - [specifically for go](#specifically-for-go)
- [packaging](#packaging)
  - [working with travis](#working-with-travis)
  - [working with appveyor](#working-with-appveyor)
  - [working with bintray](#working-with-bintray)
  - [debian](#debian)
    - [In practice](#in-practice-3)
    - [Using travis](#using-travis)
  - [RPM](#rpm)
    - [In practice](#in-practice-4)
    - [Using travis](#using-travis-1)
  - [windows](#windows)
    - [In practice](#in-practice-5)
    - [Using appveyor](#using-appveyor)
  - [github release page](#github-release-page)
- [distribution](#distribution)
- [Oops, it failed :s](#oops-it-failed-s)
- [Errors reference](#errors-reference)
- [The end !!](#the-end-)

## TLDR;

During this guide several softwares and methods are presented to produces
**ready-to-install** artifacts of your project with a two steps command:

```sh
... <apply some changes to the software>

$ changelog prepare
<manually edit the changelog>

$ gump <prerelese|patch|minor|major>
< the factory realize all the steps to
  publish the new version,
  produce the packages,
  push them to public cloud service>
```

By then end of this guie you will acquire reproducible techniques to
- maintain a changelog, with [changelog](https://github.com/mh-cbon/changelog)
- maintain a README, with [emd](https://github.com/mh-cbon/emd)
- bump your package, with [gump](https://github.com/mh-cbon/gump)
- create github release, with [gh-api-cli](https://github.com/mh-cbon/gh-api-cli)
- produce debian packages, with [go-bin-deb](https://github.com/mh-cbon/go-bin-deb)
- produce rpm packages, with [go-bin-rpm](https://github.com/mh-cbon/go-bin-rpm)
- produce windows installers, with [go-msi](https://github.com/mh-cbon/go-msi)
- produce `chocolatey` repository on `bintray`
- produce `debian` repository over `bintray`
- produce `rpm` repository over `bintray`

## head first

You might want to setup, or prepare, the required softwares on your computer.

#### a github account

Not everything you will read does require a github account, although,
this guide uses it to demonstrate a real world example,
thus, get one if you did not already signed up.

#### a bintray account

Specifically for the deployment of your artifacts on a repository,
it has `oss` support, you can sign up with your github account.

#### windows users

- Install chocolatey https://chocolatey.org/install

- Add this repository

```sh
$ choco source add -n=mh-cbon -s="https://api.bintray.com/nuget/mh-cbon/choco"
```

- Install a software

```sh
$ choco install <software> -y
$ refreshenv
```

#### debian users

- Add this repository

```sh
$ sudo add-apt-repository 'deb https://dl.bintray.com/mh-cbon/deb unstable main'
```

- Update the metadata

```sh
$ sudo apt-get -qq update
```

- Install a software

```sh
$ sudo apt-get install --allow-unauthenticated <sofware>
```

#### rpm users

- Add this repository

```sh
$ sudo curl -s -L https://bintray.com/mh-cbon/rpm/rpm > /etc/yum.repos.d/mh-cbon.repo
```

- Install a software

```sh
$ sudo dnf install <sofware> -y
```

## a dummy project

To illustrate this guide, let s create a dummy application, written in `go`, might be any language.

```sh
$ mkdir $GOPATH/src/github.com/USER/dummy
$ cd $GOPATH/src/github.com/USER/dummy
$ git init

$ cat <<EOT > src/README.md
# dummy

Says hello when invoked on the command line

## Usage

`` `sh
hello - 0.0.1

Say hello.
`` `
EOT

$ git add README.md
$ git commit README.md -m "init: add README"

$ cat <<EOT > src/main.go
// Package hello is a command line program to say hello.
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello")
}
EOT

$ git add main.go
$ git commit main.go -m "init: add main"

$ go run main.go
hello

# create teh repo in your github account
$ xdg-open https://github.com/new

# push
$ git remote add origin git@github.com:USER/dummy.git
$ git push --set-upstream origin master
```

At that point you do have a,
- a git repository with a first commit
- a git remote pointing to your github repository
- a github repository

## maintaining a changelog

[wikipedia](https://en.wikipedia.org/wiki/Changelog)

> A changelog is a log or record of all or all notable changes made to a project. The project is often a website or software project, and the changelog usually includes records of changes such as bug fixes, new features, etc. Some open source projects include a changelog as one of the top level files in their distribution.

> A changelog has historically included all changes made to a project. The "Keep a CHANGELOG" site instead advocates that a changelog not include all changes, but that it should instead contain "a curated, chronologically ordered list of notable changes for each version of a project" and should not be a "dump" of a git log "because this helps nobody".[1]

If you read the wikipedia link,
you may have noticed that a `CHANGELOG` is not defined by a file format,
instead, it is a practice to communicate to the community of users.

Now the problem is presents this way for us: in the wonderful world of software factory,
multiple tool, from multiple different vendors, with multiple different opinions, about this or that /wahtever/,
all desperately invite or enforce you to produce a changelog file in their own __incompatible__ format,
in the best case. In the worst case it is simply ignored.

To solve that situation, [changelog](https://github.com/mh-cbon/changelog) was written:
- it provides one format to rule them all
- it integrates seemlessly into a software factory process
- it simplifies and helps the developer to maintain the `CHANGELOG` file

#### In practice

For a new repository the `CHANGELOG` file must be initialized,

```sh
$ changelog init
changelog file created

$ ls -al change.log
-rw-rw-r-- ... change.log

# This is a pending change log.
$ cat change.log
UNRELEASED

  * init: add main
  * init: add README

  - USER <email>

-- USER <email>; Tue, 21 Jun 2016 11:41:13 +0200

# Add and commit this file to your repository
$ git add change.log
$ git commit change.log -m "init: changelog"
```

`changelog` generates a `change.log` file,
for each tags (or versions) existing in the repository,
it creates a new version with its associated commits in chronological order.

The `UNRELEASED` version you are seeing now, is a intermediary version containing the changes for the upcoming release.

When a new version is about to be released, changes for the next version needs to be gathered, run

```sh
$ changelog prepare
changelog file updated

$ cat change.log
UNRELEASED

  * init: changelog
  * init: add main
  * init: add README

  - USER <email>

-- USER <email>; Tue, 21 Jun 2016 11:41:13 +0200
```

This `prepare` state is here to invite to manually *update*, *fix*,
*re-arrange* the content of the changelog before it is finalized.

It is the good moment to check your commits, identify the useless commit messages,
group them around a functionnality or feature.

In this example we have various different commit messages with more or less the same meaning,
also the content of those commit are truely useless to everybody,
for clarity, lets rewrite it in this way

```
UNRELEASED

  * Initial release

  - USER <email>

-- USER <email>; Tue, 21 Jun 2016 11:41:13 +0200
```

Finally, let s manually create the first version
before we review the steps to do that via automation,

```sh
# promote UNRELEASED changes to 0.0.1
$ changelog finalize --version=0.0.1
changelog file updated

# Verify the content
$ cat change.log
0.0.1

  * Initial release

  - USER <email>

-- USER <email>; Tue, 21 Jun 2016 11:41:13 +0200

# export the change.log file to a markdwon compatible format
$ changelog md -o CHANGELOG.md --vars='{"name":"dummy"}'

$ cat CHANGELOG.md
# Changelog - dummy

### 0.0.1

__Changes__

- Initial release

__Contributors__

- USER

Released by USER, Tue 21 Jun 2016
______________

# saves those new changes
$ git add CHANGELOG.md
$ git commit CHANGELOG.md -m "init: github changelog"
$ git commit change.log -m "tag 0.0.1"

# create a tag in your vcs
$ git tag 0.0.1
$ git tag
0.0.1
$ git push --all
```

## maintain a README

https://en.wikipedia.org/wiki/README

> A README file contains information about other files in a directory or archive of computer software.
> A form of documentation, it is usually a simple plain text file called
> READ.ME, README.TXT, README.md (for a text file using markdown markup),
> README.1ST – or simply README.

The contents typically include one or more of the following:

    - Configuration instructions
    - Installation instructions
    - Operating instructions
    - A file manifest (list of files included)
    - Copyright and licensing information
    - Contact information for the distributor or programmer
    - Known bugs[2]
    - Troubleshooting[2]
    - Credits and acknowledgments
    - A changelog (usually for programmers)
    - A news section (usually for users)

Many activities to perform to maintain a `README`,

Some of those such as `installation instructions`, `contact` or `licensing` are usually a one time setup that does not change a lot.

Other sections such as `file manifest`, `usage`, `code example` are likely to change as the software evolves,
this is where maintenance help is needed.

To help with those concerns `emd` provides an engine on top of the `go` template package,
it includes features specifically designed to help with this maintenance task.

Check the detailed documentation at [emd](https://github.com/mh-cbon/emd#toc)

#### In practice

Create a new file `README.e.md` which will contain the template to generate the final `README.md`,

```sh
$ cat <<EOT > src/README.e.md
# {{.Name}}

{{pkgdoc}}

# {{toc 4}}

# Example

#### $ {{exec "go" "run" "main.go" | color "sh"}}
EOT

# commit this template to your repo for future update of the README
$ git add README.e.md
$ git commit main.go -m "init: add README"

# generate/commit the final README file
$ emd gen -in README.e.md -out README.md
$ git add README.md
$ git commit main.go -m "init: generate README"
```

That's it! The new `README.md` file  contains,

```md
  # hello

  Package hello is a command line program to say hello.

  # TOC
  - [Example](#example)

  # Example

  #### $ go run main.go
  ` ``sh
  Hello
  ` ``
```

Now it s easier to maintain the `README` file.

## bump a package

[stackoverflow](http://stackoverflow.com/a/4181188/4466350)

> Increment the version number to a new, unique value.

In [semver](http://semver.org/) semantics version are expressed such as `<major>.<minor>.<patch>-<prerelease>+<meta>`

> version numbers and the way they change convey meaning about the underlying code and what has been modified from one version to the next.

In practice changing a version number involves many side operations,

- the `CHANGELOG` needs to be updated,
- the tag needs to be created,
- the remote needs to be synced,
- the `README` might need a refresh
- linters needs to run
- tests needs to run
- and many more

[gump](https://github.com/mh-cbon/gump) was written to let you automate as many tasks as possible around that central task of version bump:
- bump using `verbs` rather than numerical values
- run pre/post operations to the version bumping

It does not compete with well-known tools like `make`, those two softwares are similar, but different,
a `make` user should keep using `make` and load it into `gump`.

### In practice

[gump](https://github.com/mh-cbon/gump) provides a very limited set of features,

```sh
$ gump -h
Gump - Bump your package

Usage:
  gump prerelease [-b|--beta] [-a|--alpha] [-d|--dry] [-m <message>]
  gump patch [-d|--dry] [-m <message>]
  gump minor [-d|--dry] [-m <message>]
  gump major [-d|--dry] [-m <message>]
```

That s all, 4 `verbs`, 4 `flags`.

However, it provides the ability to declare a file `.version.sh` to define
various hooks related to the action of version bumping:

```sh
PREBUMP=
  echo "before bumping, any verb"
PREPATCH=
  echo "before bumping to patch"
PREMINOR=
  echo "before bumping to minor"
PREMAJOR=
  echo "before bumping to major"

PREVERSION=
  echo "before bumping the version, any verb"
POSTVERSION=
  echo "after version bumped, any verb"

POSTMAJOR=
  echo "after version bumped to major"
POSTMINOR=
  echo "after version bumped to minor"
POSTPATCH=
  echo "after version bumped to patch"
POSTBUMP=
  echo "after bumping, any verb"
```

Using that file, it is now possible to define tasks to run before / after a specific kind of bump verb,

```sh
$ cat <<EOT > .version.sh
# PREBUMP runs for all kinds of bump
PREBUMP=
  # it is usefull to make sure local is in sync with remote
  git fetch --tags origin master
  git pull origin master

# PREVERSION runs for any kinds of bump, it the last pre-hook
PREVERSION=
  # use it to declare tasks that should run for any kind of bump
  go vet ./...
  go fmt ./...
  go run main.go # alike tests when you don t have tests
  # finalize the changelog
  changelog finalize --version !newversion!
  git commit change.log -m "changelog: !newversion!"
  # update the README
  emd gen -in README.e.md > README.md
  git commit README.md -m "README: !newversion!"
  # generate a markdwon version of your changelog
  changelog md -o CHANGELOG.md --vars='{"name":"dummy"}'
  git commit CHANGELOG.md -m "changelog.md: !newversion!"

# POSTVERSION runs for any kind of bumps
POSTVERSION=
  # use it to sync your local to the remote
  git push
  git push --tags
  # create a github release
  gh-api-cli create-release -n release -o USER -r dummy \
   --ver !newversion!  --draft !isprerelease! \
   -c "changelog ghrelease --version !newversion!"
EOT

$ git add .version.sh
$ git commit .version.sh -m "init: bump script"
```

Those are the steps to bump the project,

```sh
# gather your last changes
$ changelog prepare
changelog file updated

$ cat change.log
UNRELEASED

  * init: release automation

  - USER <email>

-- USER <email>; Tue, 21 Jun 2016 11:42:13 +0200

0.0.1

  * Initial release

  - USER <email>

-- USER <email>; Tue, 21 Jun 2016 11:41:13 +0200

# update the content of the UNRELEASED version
# ....

# invoke the bumper with the next version
$ gump patch
Created new tag 0.0.2
...
```

That's it, the project version `0.0.1` is patch-bumped to `0.0.2`

```sh
$ cat CHANGELOG.md
# Changelog - dummy

### 0.0.2

__Changes__

- init: release automation

__Contributors__

- USER

Released by USER, Tue 21 Jun 2016
______________

### 0.0.1

__Changes__

- Initial release

__Contributors__

- USER

Released by USER, Tue 21 Jun 2016
______________

$ git tag
0.0.2
0.0.1
```

### notes about .version.sh

It worth to note that the syntax is cross-platform,
[stackoverflow](http://stackoverflow.com/a/8055430/4466350).

[gump](https://github.com/mh-cbon/gump) takes care to handle '\\' EOL appropriately.

-  Before bumping anything, it is absolutely necessary to sync the local with your remote.

```sh
PREBUMP=
  git fetch --tags origin master
  git pull origin master
```

- For every version to bump, its good to ensure the linters/tests/compile all passes.

```sh
PREVERSION=
  go vet ./...
  go fmt ./...
  go run main.go
```

- use `POSTVERSION` to implement the setup / creation of artifacts from your computer without the help of a cloud ci.
In this example, it creates a new github release in the repo `github.com/USER/dummy`,
set it drafted if the version is a prerelease such as `beta|alpha`,
attach an extract of the version changes to the new release.

```sh
POSTVERSION=
  gh-api-cli create-release -n release -o USER -r dummy \
   --ver !newversion!  --draft !isprerelease! \
   -c "changelog ghrelease --version !newversion!"
```

## github releases

To work with github [apis](https://developer.github.com/v3/), this guide uses `gh-api-cli` a cross-platform command line utility

For a detailed documentation [visit](https://github.com/mh-cbon/gh-api-cli/).

__tokens__

For that current guide, `gh-api-cli` helps to deal with github `personal access tokens`.

Those tokens are useful to realize api calls that requires authentification.

Later you will learn how to inject those tokens into your CI files such `travis`, `appveyor`, `you name it.`

Lets create a new token with the name `release`,

```sh
$ gh-api-cli add-auth -n release -r repo
.. provides the details
```

The token is stored in a json file located at `~/gh-auths.json`.

You can retrieve token values with

```sh
$ TOKEN=`gh-api-cli get-auth -n release`
$ echo $TOKEN
```

__protip__

Sometimes it is useful to be able to upload assets by your own,

```sh
gh-api-cli upload-release-asset -t <gh token> -g <glob> -o <gh repo user> -r <gh repo name> --ver <version>
# or
gh-api-cli upload-release-asset -n <token name> -g <glob> -o <gh repo user> -r <gh repo name> --ver <version>
```

## various other useful cross-platform tools

Find various small cross-platform tools to use into your scripts,

- file pattern matching [philea](https://github.com/mh-cbon/philea)
- commit files with convenience [commit](https://github.com/mh-cbon/commit)
- improve output [666](https://github.com/mh-cbon/666)
- ignore failures [never-fail](https://github.com/mh-cbon/never-fail)
- run commands in parallel [cct](https://github.com/mh-cbon/cct)
- rm with pattern matching [rm-glob](https://github.com/mh-cbon/rm-glob)
- create `zip/tar-gz` archives [archive](https://github.com/mh-cbon/archive)

#### specifically for go

- fail the formating changed [go-fmt-fail](https://github.com/mh-cbon/go-fmt-fail)
- one-liner go build [build-them-all](https://github.com/mh-cbon/build-them-all)


## packaging

This section will guide you to create platforms specific packages.

To build platforms specific packages, the corresponding OS must be run,

You can use a [vagrant](https://www.vagrantup.com/) box,
a [docker](https://www.docker.com) image,
this guide will use [travis](http://travis-ci.org/)
and [appveyor](http://appveyor.com/)

### working with travis

Connect your `github` account to `travis` and enable your repository

```sh
xdg-open http://travis-ci.org/ # link the github account
xdg-open https://travis-ci.org/profile/USER # enable the repository
```

To encrypt a token, you can use an [online](https://travis-encrypt.github.io/) encryptor,
or the travis cli, read the doc anyway https://docs.travis-ci.com/user/encryption-keys/

To install the [travis client](https://github.com/travis-ci/travis.rb#installation),

```sh
$ gem install travis -v 1.8.2 --no-rdoc --no-ri
```

__protip__

```sh
travis encrypt --add -r USER/dummy GH_TOKEN=`gh-api-cli get-auth -n <auth name>`
```

### working with appveyor

Connect your `github` account to `appveyor` and enable your repository

```sh
xdg-open https://ci.appveyor.com/
xdg-open https://ci.appveyor.com/projects/new
```

To encrypt a token, go to

```sh
xdg-open https://ci.appveyor.com/tools/encrypt
```

__notes__: unlike travis, appveyor let you re use the same encrypted variable in every project you enable.

For advanced setup, you might find helpful to have a local vagrant running windows,
please check [this link](https://github.com/mh-cbon/go-msi/blob/master/unice-recipe.md).

### working with bintray

signup to bintray

```sh
xdg-open https://bintray.com/signup/oss
```

Open your bintray acccount, signin, then create 4 repositories:
- msi
- choco
- deb
- rpm

__token__

find your token at `https://bintray.com/profile/edit` > `API KEY`

__protip__

Delete an asset on `bintray` in a cross platform way
```sh
never-fail curl -X "DELETE" -u<USER>:<BTKEY> https://api.bintray.com/content/<USER>/choco/<file path>
```

Make sure to set this environment variable in your CI `JFROG_CLI_OFFER_CONFIG=false`

### debian

[debian wiki](https://wiki.debian.org/Packaging)

> A Debian package is a collection of files that allow for applications or libraries to be distributed via the Debian package management system.
> The aim of packaging is to allow the automation of installing, upgrading, configuring, and removing computer programs for Debian in a consistent manner.

[debian binary package](https://wiki.debian.org/Packaging/BinaryPackage)

> A Debian Package is a file that ends in .deb and contains software for your Debian system.
> A .deb is also known as a binary package. This means that the program inside the package is ready to run on your system.

The guide will demonstrate the creation of `debian binary package`.

While this is not the holy grail of debian packaging,
[go-bin-deb](https://github.com/mh-cbon/go-bin-deb) will be help you to,
- distribute your application to non developers
- install / remove your application via the standard package manager
- setup services, icons, folders etc

#### In practice

Create a `deb.json` file to describe the package content

```sh
$ cat <<EOT > deb.json
{
  "name": "dummy",
  "maintainer": "USER <email>",
  "description": "Say hello",
  "changelog-cmd": "changelog debian --vars='{\"name\":\"!name!\"}'",
  "homepage": "http://github.com/USER/!name!",
  "files": [
    {
      "from": "build/!arch!/!name!",
      "to": "/usr/bin",
      "base" : "build/!arch!/",
      "fperm": "0755"
    }
  ],
  "copyrights": [
    {
      "files": "*",
      "copyright": "2016 USER <email>",
      "license": "MIT",
      "file": "LICENSE"
    }
  ]
}
EOT
```

IE:
- create a `debian` package with the name `dummy`
- the license is `MIT` its content is available in the file `LICENSE`
- there is only one file located at `build/!arch!/dummy` to copy to `/usr/bin`
- the change list is generated with the command `changelog debian --vars='{\"name\":\"dummy\"}'`

Tokens you can find in the `json` are documented [here](https://github.com/mh-cbon/go-bin-deb#json-tokens)

Detailed documentation is available at [here](https://github.com/mh-cbon/go-bin-deb#json-file)

#### Using travis

Create a `.travis.yml` file.
It will contains the instructions to
- build the `debian` package
- upload the artifacts to a `github release`
- upload the artifacts to a `bintray`

```yml
# specific to a go project
language: go
go:
  - tip

#
env:
  matrix:
    # OKARCH is go ARCH
    # OSARCH is the package target arch
    - OKARCH=amd64 OSARCH=amd64
    - OKARCH=386 OSARCH=i386
  global:
    # rename some variable for readability
    - VERSION=${TRAVIS_TAG}
    - GH_USER=${TRAVIS_REPO_SLUG%/*}
    - GH_APP=${TRAVIS_REPO_SLUG#*/}
    # specific to bintray
    - JFROG_CLI_OFFER_CONFIG=false
    # bintray encrypted token
    - secure: yyyy


before_install:
  # install required software
  - sudo add-apt-repository 'deb https://dl.bintray.com/mh-cbon/deb unstable main'
  - sudo apt-get -qq update
  - sudo apt-get install --allow-unauthenticated changelog go-bin-deb fakeroot
  # specific to a go project
  - mkdir -p ${GOPATH}/bin
  - cd ~
  - curl https://glide.sh/get | sh

install:
  # specific to a go project
  - cd $GOPATH/src/github.com/$TRAVIS_REPO_SLUG
  - glide install
  - go install

script:
  # specific to a go project
  - go run main.go
  - go test

# create the debian package
before_deploy:
  - POOL=`echo ${GH_APP} | cut -c 1`
  # build the program to build/amd64/dummy
  - cd $GOPATH/src/github.com/$TRAVIS_REPO_SLUG
  - mkdir -p build/$OSARCH
  - GOOS=linux go build --ldflags "-X main.VERSION=$VERSION" -o build/$OSARCH/$GH_APP main.go
  # generate teh debian package to ./dummy-amd64-0.0.2.deb
  - go-bin-deb generate --file deb.json -a $OSARCH --version $VERSION -o $GH_APP-$OSARCH-$VERSION.deb
  # fetch bintray cli
  - curl -fL https://getcli.jfrog.io | sh
  # create a package of your app, ignore error if the package alreay exists
  - ./jfrog bt pc --key=$BTKEY --user=$GH_USER --licenses=MIT --vcs-url=https://github.com/$GH_USER/deb $GH_USER/deb/$GH_APP || echo "package already exists"
  # upload the artifact to the "deb" repostiory
  - ./jfrog bt upload --override=true --key $BTKEY --publish=true --deb=unstable/main/$OSARCH $GH_APP-$OSARCH-$VERSION.deb $GH_USER/deb/$GH_APP/$VERSION pool/$POOL/$GH_APP/
  # generate the repo metadata
  - curl -X POST -u ${GH_USER}:${BTKEY} https://api.bintray.com/calc_metadata/${GH_USER}/deb

# upload debian packages to the github release page
deploy:
  provider: releases
  api_key:
    secure: xxxxxxx
  file_glob: true
  file:
    - $GH_APP-$OKARCH.deb
  skip_cleanup: true
  on:
    tags: true
```

Thats it!

To trigger a travis build that generates the packages, make a new version.

```sh
$ git add deb.json .travis.yml
$ git commit -m "packaging: add debian package support"

$ changelog prepare
changelog file updated

$ changelog show
<check then improve the change list>

$ gump patch
.. commands output
Created new tag 0.0.3
.. commands output
```

Within minutes `travis` will create the packages and put them into the
[github release page](https://help.github.com/articles/creating-releases/)

### RPM

[rpm](http://www.rpm.org/)

>  The RPM Package Manager (RPM) is a powerful command line driven package management system capable of installing, uninstalling, verifying, querying, and updating computer software packages.
> Each software package consists of an archive of files along with information about the package like its version, a description, and the like.
> There is also a library API, permitting advanced developers to manage such transactions from programming languages such as C or Python.

Much like `go-bin-deb` [go-bin-rpm](https://github.com/mh-cbon/go-bin-rpm) will
help to generate a package for rpm based systems.


#### In practice

Create an `rpm.json` file to describe the package content

```sh
$ cat <<EOT > rpm.json
{
  "name": "dummy",
  "summary": "Say hello",
  "description": "A command line to say hello",
  "changelog-cmd": "changelog rpm",
  "license": "LICENSE",
  "url": "https://github.com/USER/!name!",
  "files": [
    {
      "from": "build/!arch!/!name!",
      "to": "%{_bindir}/",
      "base": "build/!arch!/",
      "type": ""
    }
  ]
}
EOT
```

- create an `rpm` package with the name `dummy`
- the license is `MIT` its content is available in the file `LICENSE`
- there is only one file located at `build/!arch!/dummy` to copy to [%{_bindir}](https://fedoraproject.org/wiki/Packaging:RPMMacros?rd=Packaging/RPMMacros#Valid_RPM_Macros) from the base path `build/!arch!/` (it will create the file `/usr/bin/dummy`)
- the change list is generated with the command `changelog rpm`

#### Using travis

Update the `.travis.yml` file to generate `rpm` packages,

```yml
# docker will run a red hat OS
sudo: required
services:
  - docker

# specific to a go project
language: go
go:
  - tip

#
env:
  matrix:
    # OKARCH is go ARCH
    # OSARCH is the package target arch
    - OKARCH=amd64 OSARCH=amd64
    - OKARCH=386 OSARCH=i386
  global:
    # rename some variable for readability
    - VERSION=${TRAVIS_TAG}
    - GH_USER=${TRAVIS_REPO_SLUG%/*}
    - GH_APP=${TRAVIS_REPO_SLUG#*/}
    # specific to bintray
    - JFROG_CLI_OFFER_CONFIG=false
    # bintray encrypted token
    - secure: yyyy

before_install:
  # update the system
  - sudo apt-get -qq update
  # specific to a go project
  - mkdir -p ${GOPATH}/bin
  - cd ~
  - curl https://glide.sh/get | sh

install:
# specific to a go project
- cd $GOPATH/src/github.com/$TRAVIS_REPO_SLUG
- glide install
- go install

script:
# specific to a go project
- go run main.go
- go test

# create the rpm package
before_deploy:
  - POOL=`echo ${GH_APP} | cut -c 1`
  # build the program to build/amd64/dummy
  - cd $GOPATH/src/github.com/$TRAVIS_REPO_SLUG
  - mkdir -p build/$OSARCH
  - GOOS=linux go build --ldflags "-X main.VERSION=$VERSION" -o build/$OSARCH/$GH_APP main.go
  # start docker
  - docker pull fedora
  # create the package in the docker
  - >
    docker run -v $PWD:/mnt/travis fedora /bin/sh -c
    "cd /mnt/travis && \
    (curl -s -L https://bintray.com/mh-cbon/rpm/rpm > /etc/yum.repos.d/w.repo) && \
     dnf install go-bin-rpm changelog rpm-build -y --quiet && \
     go-bin-rpm generate --file rpm.json -a $OSARCH --version $VERSION -o $GH_APP-$OSARCH-$VERSION.rpm"
  # create a package of your app, ignore error if the package alreay exists
  - ./jfrog bt pc --key=$BTKEY --user=$GH_USER --licenses=MIT --vcs-url=https://github.com/$GH_USER/rpm $GH_USER/rpm/$GH_APP || echo "package already exists"
  # upload to bintray
  - ./jfrog bt upload --override=true --key $BTKEY --publish=true $GH_APP-$OSARCH-$VERSION.rpm $GH_USER/rpm/$GH_APP/$VERSION pool/$POOL/$GH_APP/
  # generate the repo metadata
  - curl -X POST -u ${GH_USER}:${BTKEY} https://api.bintray.com/calc_metadata/${GH_USER}/rpm

# upload rpm packages to the github release page
deploy:
  provider: releases
  api_key:
    secure: xxxx
  file_glob: true
  file:
    - $GH_APP-$OKARCH.rpm
  skip_cleanup: true
  on:
    tags: true
```

Thats it!

To trigger a travis build that generates the packages, make a new version.

```sh
$ git add rpm.json docker.sh
$ git commit -m "packaging: add rpm package support"

$ changelog prepare
changelog file updated

$ changelog show
<check then improve the change list>

$ gump patch
.. commands output
Created new tag 0.0.4
.. commands output
```

Within minutes `travis` will create the packages and put them into the
[github release page](https://help.github.com/articles/creating-releases/)


### windows

[wikipedia](https://en.wikipedia.org/wiki/WiX)

> The Windows Installer XML Toolset (WiX, pronounced "wicks"), is a free software toolset that builds Windows Installer packages from XML code.
> It consists of a command-line environment that developers may integrate into their build processes to build MSI and MSM packages.

[wix documentation](http://wixtoolset.org/documentation/manual/v3/)

In the same spirit as `go-bin-deb` and `go-bin-rpm`, [go-msi](https://github.com/mh-cbon/go-msi) will help you to produce `msi` package and `nuget` packages.

`nuget` packages are useful to integrate with [chocolatey](https://chocolatey.org/)

#### In practice

Create a file `wix.json` to describe the package content

```sh
$ cat <<EOT > wix.json
{
  "product": "dummy",
  "company": "USER",
  "license": "LICENSE",
  "upgrade-code": "",
  "files": {
    "guid": "",
    "items": [
      "dummy.exe"
    ]
  },
  "env": {
    "guid": "",
    "vars": [
      {
        "name": "PATH",
        "value": "[INSTALLDIR]",
        "permanent": "no",
        "system": "no",
        "action": "set",
        "part": "last"
      }
    ]
  }
}
EOT
```

- create an `msi` installer containing `dummy.exe`
- copy the file at ~~ `C:\program files\dummy`
- add `C:\program files\dummy` to the `PATH`

Then setup `GUIDs` value within the `wix.json` file with the help of `go-msi`, run

```sh
$ go-msi set-guid
file updated
```

__notes__: guid s should be set once at beginning and not updated later.

#### Using appveyor

To perform the build of the package, create an `appveyor.yml` file.

```yml
image: Visual Studio 2017
clone_folder: c:\gopath\src\github.com\%APPVEYOR_REPO_NAME%

# set to false to build on any commit.
skip_non_tags: true

environment:
  # improve readability
  VCS_URL: https://github.com/%APPVEYOR_REPO_NAME%
  GH_APP: %APPVEYOR_PROJECT_NAME%
  GH_USER: %APPVEYOR_ACCOUNT_NAME%
  VERSION: %APPVEYOR_REPO_TAG_NAME%
  # specific to go
  GOPATH: c:\gopath
  # specific to bintray
  JFROG_CLI_OFFER_CONFIG: false
  #  define secure tokens
  CHOCOKEY:
    secure: xxx
  BT_KEY:
    secure: yyy
  GH_TOKEN:
    secure: zzz

# prepare system and project
install:
  # install required software
  - choco source add -n=mh-cbon -s="https://api.bintray.com/nuget/mh-cbon/choco"
  - choco install changelog gh-api-cli go-msi -y
  - refreshenv
  # to test the build on non tag commits.
  - if "%x%"=="%VERSION%" set VERSION=1.0.2
  # ensure wix is available in PATH
  - set PATH=%WIX%\bin;%PATH%
  # fetch bintray client
  - curl -fsSk -o jfrog.exe -L "https://api.bintray.com/content/jfrog/jfrog-cli-go/$latest/jfrog-cli-windows-amd64/jfrog.exe?bt_package=jfrog-cli-windows-amd64"
  # specific to go
  - set PATH=%GOPATH%\bin;%PATH%
  - go get -u github.com/mh-cbon/never-fail
  - go get -u github.com/Masterminds/glide
  - glide install

test_script:
  - go test
  - go run main.go

# build msi artifacts
build_script:
  # build the program for x386 arch
  - set GOARCH=386
  - go build -o %GH_APP%.exe --ldflags "-X main.VERSION=%VERSION%" main.go
  # generate the x386 MSI package
  - go-msi make --msi %GH_APP%-%GOARCH%-%VERSION%.msi --version %VERSION% --arch %GOARCH%
  # build a cross platform x386 nuget package.
  - go-msi choco --path wix.json --version %VERSION% --input %GH_APP%-%GOARCH%-%VERSION%.msi --changelog-cmd "changelog ghrelease --version %VERSION%"
  # build the program for amd64 arch
  - set GOARCH=amd64
  - go build -o %GH_APP%.exe --ldflags "-X main.VERSION=%VERSION%" main.go
  # generate the amd64 MSI package
  - go-msi make --msi %GH_APP%-%GOARCH%-%VERSION%.msi --version %VERSION% --arch %GOARCH%

# configure the artifacts to upload to the github releae page
artifacts:
  - path: '*-386.msi'
    name: msi-x86
  - path: '*-amd64.msi'
    name: msi-x64

# setup the github release page upload
deploy:
  - provider: GitHub
    artifact: msi-x86, msi-x64
    force_update: false
    draft: false
    prerelease: false
    description: "Release %APPVEYOR_REPO_TAG_NAME%"
    auth_token:
      secure: xxxx
    on:
      appveyor_repo_tag: true

# deploy the nuget/msi packages to bintray
deploy_script:
  # setup both "choco" and "msi" repositories on bintray
  - never-fail jfrog bt pc --user %GH_USER% --key %BT_KEY% --licenses=MIT --vcs-url=https://github.com/%APPVEYOR_REPO_NAME% %GH_USER%/msi/%GH_APP%
  - never-fail jfrog bt pc --user %GH_USER% --key %BT_KEY% --licenses=MIT --vcs-url=https://github.com/%APPVEYOR_REPO_NAME% %GH_USER%/choco/%GH_APP%
  # upload the nupkg
  - jfrog bt upload --user %GH_USER% --key %BT_KEY%  --override=true --publish=true %GH_APP%.%VERSION%.nupkg %GH_USER%/choco/%GH_APP%/%VERSION%
  # upload the x386 msi artifact
  - set GOARCH=386
  - jfrog bt upload --user %GH_USER% --key %BT_KEY%  --override=true --publish=true %GH_APP%-%GOARCH%-%VERSION%.msi %GH_USER%/msi/%GH_APP%/%VERSION%
  # upload the amd64 msi artifact
  - set GOARCH=amd64
  - jfrog bt upload --user %GH_USER% --key %BT_KEY%  --override=true --publish=true %GH_APP%-%GOARCH%-%VERSION%.msi %GH_USER%/msi/%GH_APP%/%VERSION%
```

Thats it!

To trigger an appveyor build that generates the packages, make a new version.

```sh
$ git add wix.json appveyor.yml
$ git commit -m "packaging: add msi package support"

$ changelog prepare
changelog file updated

$ changelog show
<check then improve the change list>

$ gump patch
.. commands output
Created new tag 0.0.5
.. commands output
```

Within minutes both `appveyor` and `travis` will create the packages
and put all of them into the
[github release page](https://help.github.com/articles/creating-releases/)

### github release page

Sometimes you want to distribute only build artifacts and not the whole packages and repostiory.

$ .version.sh
```sh
POSTVERSION=
  # clean up
  666 rm-glob -r build/
  666 rm-glob -r assets/
  # build program for many OS-ARCH
  666 build-them-all build main.go -o "build/&os-&arch/&pkg" --ldflags "-X main.VERSION=!newversion!"
  # create a ZIP archive for windows
  philea -s -S -p "build/windows*/*" "666 archive create -f -o=assets/%dname.zip -C=build/%dname/ ."
  # create a TGZ archive for !windows
  philea -s -S -e windows*/** -p "build/*/**" "666 archive create -f -o=assets/%dname.tar.gz -C=build/%dname/ ."
  # create a github release
  666 gh-api-cli create-release -n release -o USER -r dummy --ver !newversion! -c "changelog ghrelease --version !newversion!" --draft !isprerelease!
  # upload assets to the release
  666 gh-api-cli upload-release-asset -n release --glob "assets/*" -o USER -r dummy --ver !newversion!
  # clean up
  666 rm-glob -r build/
  666 rm-glob -r assets/
```

# distribution

You are all set now!

You can now add those instructions to your `README`,

__windows__

```sh
 - choco source add -n=<USER> -s="https://api.bintray.com/nuget/<USER>/choco"
 - choco install dummy -y
 - refreshenv
```

__debian__

```sh
 - sudo add-apt-repository 'deb https://dl.bintray.com/<USER>/deb unstable main'
 - sudo apt-get -qq update
 - sudo apt-get install --allow-unauthenticated -y dummy
```

__redhat__

```sh
- sudo curl -s -L https://bintray.com/<USER>/rpm/rpm > /etc/yum.repos.d/<USER>.repo
- sudo dnf install go-bin-rpm changelog rpm-build -y
```

__protip__

`emd` has helpers for that matter, [see here](https://github.com/mh-cbon/emd#std).

# Oops, it failed :s

For various reasons the build might fail, on bump or during the ci,
in such case you are left with a project in a loosy state.

For example, you might end up with,
- a git tag
- a gh release
- failed ci did not produce packages

The simplest solution is to simply remove traces of that broken release,
fix the bump pipeline, then bump again.

```sh
$ gump patch
...
failure
....
$ VERSION=x.x.x
# revert, fix then restart
$ git tag -d $VERSION
$ git push origin :$VERSION
$ gh-api-cli rm-release -n <auth name> -o USER -r dummy --ver $VERSION
# rename last release to UNRELEASED
$ changelog rename
# apply fixes to the pipeline
$ gump patch # again
```

# Errors reference

Reports your difficulties to this repository to help to build this index of errors reference.

# The end !!

This is it, with some tools and cloud services,
the releases of the software are kept up to date with a two step commands,

```sh
$ changelog prepare
$ gump <patch|minor|major>
```

Enjoy the reusable and modular system those tools provide you to feet your own
vision, contribute and report your success.

~~ Happy coding!
