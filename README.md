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

...
```

The `bin/detect` script is run upon the provided application source folder, and we expect `Count` to exist in the root of this folder. If not, then the `display-count` buildpack cannot help this application.

### bin/build

The other required executable for every buildpack is `bin/build`. It can add a layer of software to the final image, can setup the runtime environment of running containers, can depend upon other buildpacks, and can propose a command to run when the OCI is launched.

Our `buildpacks/fizzbuzz-standalone/bin/build` is written to just copy a set of files into a new layer of the final runnable OCI/docker image. The three important lines are:

```bash
layer_dir=$1
buildpack_dir="$( cd "$( dirname "${BASH_SOURCE[0]}" )/.." && pwd )"
cp -r $buildpack_dir/layer/* $layer_dir/
```

TODO: [What about the application folder?] The "layer" directory, passed to `bin/build` as the first argument, is the only folder area that `bin/build` should modify.

The files placed into `$layer_dir` are copied from the buildpack's `layer` folder:

```plain
├── fizzbuzz
│   └── bin
│       └── display-count
├── fizzbuzz.toml
└── launch.toml
```

The `display-count` executable will be automatically available in the `$PATH`, as a layer's `<name>/bin` folder (`$layers/fizzbuzz/bin` above) will automatically be added to `$PATH` of each running container.

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

Finally, we need to tell `pack build` that our `$layers` changes should be included in the final OCI/docker image. We set the `launch = true` attribute in `fizzbuzz.toml`:

```toml
launch = true
```

## Test buildpacks without builder

```plain
$ cat fixtures/fifteen/Count
15
$ pack build playtime --buildpack buildpacks/fizz --buildpack buildpacks/buzz --path fixtures/fifteen
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

$ docker run -ti playtime ls -al
Count  buzz  fizz
```

```plain
$ cat fixtures/three/Count
3
$ pack build playtime --buildpack buildpacks/fizz --path fixtures/three
===> DETECTING
[detector] ======== Results ========
[detector] pass: com.starkandwayne.buildpacks.playtime.fizz@1.0.0
[detector] Resolving plan... (try #1)
[detector] Success! (1)
...
===> BUILDING
[builder] ---> Fizz Buildpack
[builder] [[entries]]
[builder]   name = "fizz"
[builder]   version = ""
[builder] ---> Create fizz file

$ docker run -ti playtime ls -al
Count  fizz
```

## Builder

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
