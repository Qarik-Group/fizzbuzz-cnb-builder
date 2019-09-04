# Solving FizzBuzz with Cloud Native Buildpacks

In this project and README I'm going to introduce you to Cloud Native Buildpacks using the silliest method for solving FizzBuzz. Yeah.

## FizzBuzz Coding Challenge

Write a program that takes a sequence of numbers, 1, 2, 3. If the number is a multiple of three print "fizz", if the number is a multiple of five print "buzz", if its both print "fizzbuzz", or if its neither then print the number.

The output might look like: 1, 2, fizz, 4, buzz, 6, ...

## Cloud Native Buildpacks

Buildpacks are a decade-old method for converting application source code or artifacts into a running application container image that originated at Heroku and the Cloud Foundry project.

Cloud Native Buildpacks are the merging of lessons learn from Heroku and Cloud Foundry to bring buildpacks to everyone - Docker, Kubernetes, and more.

A NodeJS application with its `package.json`, and either `package-lock.json` or `yarn.lock` files, will be automatically detected as such, have NodeJS installed, have the npm libraries downloaded, and the resulting image be ready to run anywhere that Docker images can be launched.

## Solving FizzBuzz with Buildpacks

Ok, this is were I abuse the FizzBuzz problem. Here's how we're going to solve this whilst using and explaining Cloud Native Buildpacks (CNBs, or buildpacks, for short).

Consider an application with a `Count` file containing an integer. When we "build" our application -- applying one or more CNBs to it -- the resulting image will print out that `Count` value and exit successfully.

But, if the `Count` value is a multiple of 3, then instead of printing the number we want it to print `fizz`. Similarly, if the number is a multiple of 5 then print `buzz`. Or both. But no number.

Instead of determining to print a number, `fizz`, `buzz`, or `fizzbuzz` at runtime, we will make the decision at the time we build the application image. What will be printed will be hard baked into the image based on the source `Count` file.

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

In each example our trivial source code containing only `Count` file has become a runnable Open Container Image (OCI), or Docker image.

That's the power of CNBs -- to convert your source code into a runnable application. In the example above it is a short-lived application that prints a number or word. Commonly, CNBs are used on long-lived web applications. Either is good.

### Introducing our FizzBuzz buildpacks

I've solved this problem with three CNBs in order to explore multi-buildpack support, and how optional buildpacks can be used.

* `buildpacks/display-count` is the primary buildpack that produces a runnable OCI, if source code contains a `Count` file
* `buildpacks/fizz` is a buildpack that detects if there is a `Count` file, and if its contents are a multiple of 3
* `buildpacks/buzz` is a buildpack that detects if there is a `Count` file, and if its contents are a multiple of 5

Let's try out `display-count` and see that it can convert any of our `fixtures` applications, each with a `Count` file, into a running application that merely prints out the `Count` value.

Some of the output is included below for discussion:

```plain
$ pack build playtime \
    --builder cloudfoundry/cnb:bionic \
    --buildpack buildpacks/display-count \
    --path fixtures/fifteen
Selected run image cloudfoundry/run:base-cnb
fetching buildpack from buildpacks/display-count
adding buildpack com.starkandwayne.buildpacks.playtime.display-count version 1.0.0 to builder
Executing lifecycle version 0.4.0
===> DETECTING
[detector] ======== Results ========
[detector] pass: com.starkandwayne.buildpacks.playtime.display-count@1.0.0
[detector] Resolving plan... (try #1)
[detector] Success! (1)
...
===> BUILDING
[builder] ---> Display FizzBuzz Count Buildpack
[builder] ---> Calculating fizz and buzz
[builder]      No fizz, no buzz, all number
[builder] ---> Create launch process to display count on start
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
