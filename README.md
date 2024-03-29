# Solving FizzBuzz with Cloud Native Buildpacks

In this project and README I'm going to introduce you to Cloud Native Buildpacks using the silliest method for solving FizzBuzz. Yeah.

This repo and tutorial are compliamentary to the official [Creating a Cloud Native Buildpack](https://buildpacks.io/docs/create-buildpack/) walk thru that is also highly recommended.

## FizzBuzz Coding Challenge

Write a program that takes a sequence of numbers, 1, 2, 3. If the number is a multiple of three print "fizz", if the number is a multiple of five print "buzz", if its both print "fizzbuzz", or if its neither then print the number.

The output might look like: 1, 2, fizz, 4, buzz, 6, ...

## Cloud Native Buildpacks

Buildpacks are a decade-old method for converting application source code or artifacts into a running application container image that originated at Heroku and the Cloud Foundry project.

Cloud Native Buildpacks are the merging of lessons learn from Heroku and Cloud Foundry to bring buildpacks to everyone - Docker, Kubernetes, and more.

A NodeJS application with its `package.json`, and either `package-lock.json` or `yarn.lock` files, will be automatically detected as such, have NodeJS installed, have the npm libraries downloaded, and the resulting image be ready to run anywhere that Docker images can be launched.

## Solving FizzBuzz with a Buildpack

Ok, this is were I abuse the FizzBuzz problem. Here's how we're going to solve this whilst using and explaining Cloud Native Buildpacks (CNBs, or buildpacks, for short).

Consider an application with a `Count` file containing an integer. When we "build" our application -- applying one or more CNBs to it -- the resulting image will print out that `Count` value and exit successfully.

But, if the `Count` value is a multiple of 3, then instead of printing the number we want it to print `fizz`. Similarly, if the number is a multiple of 5 then print `buzz`. Or both. But no number.

Instead of determining to print a number, `fizz`, `buzz`, or `fizzbuzz` at runtime, we will make the decision at the time we build the application image. What will be printed will be hard baked into the image based on the source `Count` file.

### Installing `pack` CLI

In time you'll learn there are multiple ways to use Cloud Native Buildpacks to convert your source code into runnable application images. In this README we'll only use the `pack` CLI.

Instructions for installation are at https://buildpacks.io/docs/install-pack/

NOTE: you need `pack` 0.4.0 or pack from source.

### Simple demo

Here is a working example that you can run now:

```plain
mkdir -p ~/workspace/fizzbuzz-app
cd ~/workspace/fizzbuzz-app

echo 1 > Count
pack build fizzbuzz-app --builder starkandwayne/fizzbuzz-builder
docker run fizzbuzz-app
# => 1

echo 3 > Count
pack build fizzbuzz-app --builder starkandwayne/fizzbuzz-builder --no-pull
docker run fizzbuzz-app
# => fizz

echo 5 > Count
pack build fizzbuzz-app --builder starkandwayne/fizzbuzz-builder --no-pull
docker run fizzbuzz-app
# => buzz

echo 15 > Count
pack build fizzbuzz-app --builder starkandwayne/fizzbuzz-builder --no-pull
docker run fizzbuzz-app
# => fizzbuzz
```

In each example above, the trivial source code contains only a `Count` file, but is convert into a runnable Open Container Image (OCI), or Docker image.

That's the power of CNBs -- to convert your source code into a runnable application. In the example above it is a short-lived application that prints a number or word. Commonly, CNBs are used on long-lived web applications. Either is good.

### Introducing our FizzBuzz buildpacks

I've solved this problem twice.

The first time is with a single buildpack `buildpacks/fizzbuzz-standalone` that produces a runnable OCI which solves the fizzbuzz problem for the value in `Count`.

The second solution explores CNB multi-buildpack support, and how optional buildpacks can be used:

* `buildpacks/display-count` is the primary buildpack that produces a runnable OCI, if source code contains a `Count` file
* `buildpacks/fizz` is a buildpack that detects if there is a `Count` file, and if its contents are a multiple of 3
* `buildpacks/buzz` is a buildpack that detects if there is a `Count` file, and if its contents are a multiple of 5

### All-in-one fizzbuzz-standalone

Let's try out `buildpacks/fizzbuzz-standalone` and see that it can convert any of our `fixtures` test applications, each with a `Count` file, into a running application that solves the FizzBuzz problem for the value in the `Count` file of the source application.

Some of the output is included below for discussion:

```plain
$ pack build playtime \
    --builder cloudfoundry/cnb:bionic \
    --buildpack buildpacks/fizzbuzz-standalone \
    --path fixtures/fifteen
...
Selected run image cloudfoundry/run:base-cnb
fetching buildpack from buildpacks/fizzbuzz-standalone
adding buildpack com.starkandwayne.buildpacks.fizzbuzz-standalone version 1.0.0 to builder
Executing lifecycle version 0.4.0
===> DETECTING
[detector] ======== Results ========
[detector] pass: com.starkandwayne.buildpacks.fizzbuzz-standalone@1.0.0[detector] Resolving plan... (try #1)
[detector] Success! (1)
...
===> BUILDING
[builder] ---> FizzBuzz Buildpack
[builder] ---> Copy files from buildpack
===> EXPORTING
...
[exporter] *** Images:
[exporter]       index.docker.io/library/playtime:latest - succeeded
...
```

The local `playtime` image is runnable and displays the `Count` value from the `fixtures/fifteen` source folder:

```plain
$ docker run playtime
15
```

Looking back at the output from `pack build` we see three stages: `DETECTING`, `BUILDING`, and `EXPORTING`. In the actual output there is also `RESTORING`, and `ANALYZING`, which we will not cover in this tutorial.

The **Detecting** stage allows each buildpack (we only provided one in our example) to determine if it can perform any service or provide any assistance to this application.

Each buildpack implements this with a `bin/detect` executable.

The **Building** stage allows the selected buildpacks to inject a layer into the final OCI image, and to propose how the final runnable image should run.

Each buildpack implements this with a `bin/build` executable.

The **Exporting** stage generates an OCI, adds metadata, and optionally publishes to a registry for remote systems to use.

The `playtime` image contains an entrypoint, tags, and metadata from the `pack build` exporting stage:

```plain
$ docker inspect playtime
...
  "Entrypoint": [
      "/cnb/lifecycle/launcher"
  ],
  "Labels": {
      "io.buildpacks.build.metadata": "{...}",
      "io.buildpacks.lifecycle.metadata": "{...}",
      "io.buildpacks.stack.id": "io.buildpacks.stacks.bionic"
  }
...
```

### `bin/detect`

The first require executable of every CNB is `bin/detect`. It determines if this buildpack can be helpful to the provided application source code.

If we ran the `pack build --buildpack buildpacks/fizzbuzz-standalone` command upon another folder that didn't have a `Count` file then the buildpack would alert and fail:

```plain
$ pack build playtime \
    --builder cloudfoundry/cnb:bionic \
    --buildpack buildpacks/fizzbuzz-standalone \
    --path .
...
===> DETECTING
[detector] ======== Output: com.starkandwayne.buildpacks.fizzbuzz-standalone@1.0.0 ========
[detector] No Count file
[detector] Error: failed to detect: detection failed
[detector] ======== Results ========
[detector] err:  com.starkandwayne.buildpacks.fizzbuzz-standalone@1.0.0 (1)
```

The output includes an explanation `No Count file`. Indeed the local folder (from `--path .`) does not have a `Count` file.

The `buildpacks/fizzbuzz-standalone/bin/detect` file shows how we did this with a shell script:

```bash
#!/bin/bash

[[ -f Count ]] || { echo "No Count file"; exit 1; }
```

If the current directory -- the application source code -- does not have a `Count` file then print an error message and exit with code 1.

### bin/build

The other required executable for every buildpack is `bin/build`. It can add a layer of software to the final image, can setup the runtime environment of running containers, can depend upon other buildpacks, and can propose a command to run when the OCI is launched.

Our `buildpacks/fizzbuzz-standalone/bin/build` is written to just copy a set of files into the final runnable OCI/docker image.

```bash
cp -r $buildpack_dir/layer/* $layer_dir/
```

Many buildpacks will unpack large assets, such as programming language compilers or intepreters, into the `$layer_dir`.

In our `bin/build` the files placed into `$layer_dir` are copied from the buildpack's `layer` folder:

```plain
├── fizzbuzz
│   └── bin
│       └── display-count
├── fizzbuzz.toml
└── launch.toml
```

The `display-count` executable will be automatically available in the `$PATH`, as a layer's `<name>/bin` folder (`$layer_dir/fizzbuzz/bin` above) will automatically be added to `$PATH` of each running container.

We can confirm that our previously built `playtime` image does indeed include a `display-count` executable that's in the `$PATH`, by explicitly running it:

```bash
$ docker run playtime display-count
15
```

We want `display-count` executable to be run automatically when our final OCI/docker image is run, so we declare a "web" process with the `launch.toml` file:

```toml
[[processes]]
command = "display-count"
type = "web"
```

Finally, we need to tell `pack build` that our `$layer_dir` changes should be included in the final OCI/docker image. We set the `launch = true` attribute in `fizzbuzz.toml`:

```toml
launch = true
```

## Builder

NOTE: `pack create-builder` requires `pack` v0.4.0+ or built from source. v0.3.0 will not work.

We can distribute our buildpack and make it readily available to anyone by creating a Builder.

We first used a pre-existing public Builder in the initial demonstration of `pack build`:

```plain
pack build fizzbuzz-app --builder starkandwayne/fizzbuzz-builder
```

The Builder contains one or more buildpacks, and hopefully one of them can apply itself to the application source code.

Some Builders contain dozens of buildpacks, such as the Cloud Foundry `cloudfoundry/cnb:cflinuxfs3`. To see what buildpacks are available in a Builder use `pack inspect-builder`:

```plain
pack inspect-builder cloudfoundry/cnb:cflinuxfs3
```

We will create a new Builder that only contains our `fizzbuzz-standalone` buildpack.

The `builder-standalone.toml` describes our Builder.

It specifies that only one buildpack be included, and the path to the buildpack:

```toml
[[buildpacks]]
  id = "com.starkandwayne.buildpacks.fizzbuzz-standalone"
  uri = "buildpacks/fizzbuzz-standalone"
```

The additional metadata, including version, for the buildpack are discovered from the buildpack's source folder or tarball.

Whilst there is only one buildpack in our Builder, we need to describe it explicitly in a Builder Order Group:

```toml
[[order]]
[[order.group]]
  id = "com.starkandwayne.buildpacks.fizzbuzz-standalone"
```

We'll visit `[[order.group]]` later when we solve FizzBuzz again using multiple buildpacks.

Finally, all our buildpacks require a common run image and build image:

```toml
[stack]
  id = "io.buildpacks.stacks.bionic"
  build-image = "cloudfoundry/build:base-cnb"
  run-image = "cloudfoundry/run:base-cnb"
```

At the time of writing, the Ubuntu Bionic-based images above (`:base-cnb` => Ubuntu Bionic) included the `yj` CLI, whereas `cflinuxfs3` images did not. They are also smaller images than `cflinuxfs3`-based images.

Our runtime fizzbuzz application has very few dependencies -- `bash` shell -- but the build sequences (the `bin/detect` and `bin/build`) require `bash`, `jq`, and `yj`.

### Create a Builder image

```plain
pack create-builder \
  starkandwayne/fizzbuzz-standalone-builder \
  -b builder-standalone.toml
```

Once completed, our local Builder image `starkandwayne/fizzbuzz-standalone-builder` can be used to build our fixture applications:

```plain
pack build playtime --builder starkandwayne/fizzbuzz-standalone-builder \
  --path fixtures/fiften
docker run playtime
# => fizzbuzz

pack build playtime --builder starkandwayne/fizzbuzz-standalone-builder \
  --path fixtures/five
docker run playtime
# => buzz

pack build playtime --builder starkandwayne/fizzbuzz-standalone-builder \
  --path fixtures/one
docker run playtime
# => 1
```

## Additional Buildpack, Same Builder

Let's look at new buildpack `print-message`, unrelated to FizzBuzz, that activates if an application contains a `Message` file, and when running it displays the contents of the file.

We'll use this `print-message` buildpack to introduce some new concepts:

* Process types
* Setting up the environment
* Two buildpacks in same Builder

### Explore print-message buildpack

### Two process types

This buildpack offers two process types `web` and `task`.

```toml
[[processes]]
type = "web"
command = "display-message"

[[processes]]
type = "task"
command = "show-message-twice"
```

As explained `display-message` executable prints the application's `Message`. The `show-message-twice` executable... prints the `Message` twice...

By default, a `web` process type will be run:

```plain
$ docker run playtime
hello world
```

We can run the image with one of the alternate process types `task` in multiple ways:

```plain
docker run -e CNB_PROCESS_TYPE=task playtime
docker run playtime task
```

The `$CNB_PROCESS_TYPE` method is more future-proof and less susceptible to accidental errors with the latter. If a process type `task` exists, then it is run. If not, then it will attempt to run an executable from `$PATH` called `task`.

Being explicit with `$CNB_PROCESS_TYPE` means you can a fast error if you request a missing process type, rather than accidentally run a command that exists:

```plain
$ docker run -e CNB_PROCESS_TYPE=date playtime
Error: failed to launch: determine start command: process type date was not found
$ docker run playtime date
Thu Sep  5 02:08:57 UTC 2019
```

### Setting up the environment

Each buildpack can install shell scripts into its layer that will be run on start, prior to the process type (`web` or `task` above) or any other command is launched.

To see this in action, our `print-message`-based `playtime` image from above has a `$MESSAGE` environment variable created during startup:

```plain
$ docker run playtime env
...
MESSAGE=hello world
```

The buildpack achieved this feat by installing `$layer_dir/print-message/profile.d/message.sh`. Any script in `profile.d` will be loaded into the environment before the container's primary process is launched.

These `profile.d` scripts can be used to setup environment variables, create new configuration files, and more.

### Two buildpacks in same Builder

We now have two buildpacks for two different type of source code - one creates a runnable image from a `Count` file, the other creates a runnable image from a `Message` file.

We can create a single Builder with both buildpacks and it will automatically detect which buildpack to use for each type of source code.

```plain
pack create-builder starkandwayne/fizzbuzz-printmessage-builder -b builder-fizzbuzz-printmessage.toml
```

If we use our Builder against a `Count` application we get the fizzbuzz output:

```plain
pack build playtime --no-pull --builder starkandwayne/fizzbuzz-challenge-builder --path fixtures/fifteen
docker run playtime
# => fizzbuzz
```

If we use our Builder against a `Message` application we see the message printed:

```plain
pack build playtime --builder starkandwayne/fizzbuzz-challenge-builder --path fixtures/hello-world
docker run playtime
# => hello world
```

The `builder-fizzbuzz-printmessage.toml` file describes the combined Builder:

* contains two `[[buildpacks]]` to ensure the two buildpacks are bundled into the Builder image
* contains two `[[order]]` sections so that only one or the other buildpack is applied to any given application

If an application contained both a `Count` and `Message` then the first `[[order]]` would win and that group of buildpacks (only one buildpack in each group here) would be applied to the application.

The application `fixtures/fiften` contains both, and since the `[[order]]` containing `fizzbuzz-standalone` buildpack is first, it is the only buildpack (group) that is applied to the application:

```plain
$ pack build playtime --builder starkandwayne/fizzbuzz-challenge-builder --path fixtures/fifteen
...
===> DETECTING
[detector] ======== Results ========
[detector] pass: com.starkandwayne.buildpacks.fizzbuzz-standalone@1.0.0
[detector] Resolving plan... (try #1)
[detector] Success! (1)
===> BUILDING
[builder] ---> FizzBuzz Buildpack
```

We will learn even more soon about `[[order]]` when we look at multiple buildpacks per `[[order]]`.

## Multi-buildpack Builder

NOTE: `pack create-builder` requires `pack` v0.4.0+ or built from source. v0.3.0 will not work.

### Create builder with buildpacks

```plain
pack create-builder starkandwayne/fizzbuzz-builder -b builder.toml
```

Test the various fixture apps on the builder.

The app with `Count` as a multiple of 3 and 5 will see both buildpacks detected and used:

```plain
$ cat fixtures/one/Count
15
$ pack build playtime --builder starkandwayne/fizzbuzz-builder --no-pull --path fixtures/fifteen
===> DETECTING
[detector] ======== Results ========
[detector] pass: com.starkandwayne.buildpacks.playtime.fizz@1.0.0
[detector] pass: com.starkandwayne.buildpacks.playtime.buzz@1.0.0
[detector] Resolving plan... (try #1)
[detector] Success! (2)
...
===> BUILDING
[builder] ---> Fizz Buildpack
[builder] [[entries]]
[builder]   name = "fizz"
[builder]   version = ""
[builder] ---> Create fizz file
[builder] ---> Buzz Buildpack
[builder] [[entries]]
[builder]   name = "buzz"
[builder]   version = ""
[builder] ---> Create buzz file
```

With `Count` equal to five, the `fizz` buildpack will fail detection:

```plain
$ cat fixtures/one/Count
5
$ pack build playtime --builder starkandwayne/fizzbuzz-builder --no-pull --path fixtures/five

===> DETECTING
[detector] ======== Output: com.starkandwayne.buildpacks.playtime.fizz@1.0.0 ========
[detector] Count not a multiple of 3
[detector] ======== Results ========
[detector] err:  com.starkandwayne.buildpacks.playtime.fizz@1.0.0 (103)
[detector] pass: com.starkandwayne.buildpacks.playtime.buzz@1.0.0
[detector] Resolving plan... (try #1)
[detector] Success! (1)
...
===> BUILDING
[builder] ---> Buzz Buildpack
[builder] [[entries]]
[builder]   name = "buzz"
[builder]   version = ""
[builder] ---> Create buzz file
```

With `Count` equal to three, the `buzz` buildpack will fail detection:

```plain
$ cat fixtures/one/Count
3
$ pack build playtime --builder starkandwayne/fizzbuzz-builder --no-pull --path fixtures/three
===> DETECTING
[detector] ======== Output: com.starkandwayne.buildpacks.playtime.buzz@1.0.0 ========
[detector] Count not a multiple of 5
[detector] ======== Results ========
[detector] pass: com.starkandwayne.buildpacks.playtime.fizz@1.0.0
[detector] err:  com.starkandwayne.buildpacks.playtime.buzz@1.0.0 (105)
[detector] Resolving plan... (try #1)
[detector] Success! (1)
...
===> BUILDING
[builder] ---> Fizz Buildpack
[builder] [[entries]]
[builder]   name = "fizz"
[builder]   version = ""
[builder] ---> Create fizz file
```

And if the `Count` is neither a multiple of 3 nor 5, then neither buildpack succeeds:

```plain
$ cat fixtures/one/Count
1
$ pack build playtime --builder starkandwayne/fizzbuzz-builder --no-pull --path fixtures/one
===> DETECTING
[detector] ======== Output: com.starkandwayne.buildpacks.playtime.fizz@1.0.0 ========
[detector] Count not a multiple of 3
[detector] ======== Output: com.starkandwayne.buildpacks.playtime.buzz@1.0.0 ========
[detector] Count not a multiple of 5
[detector] ======== Results ========
[detector] err:  com.starkandwayne.buildpacks.playtime.fizz@1.0.0 (103)
[detector] err:  com.starkandwayne.buildpacks.playtime.buzz@1.0.0 (105)
[detector] Resolving plan... (try #1)
[detector] fail: no viable buildpacks in group
[detector] Error: failed to detect: detection failed
```
