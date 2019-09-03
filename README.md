# FizzBuzz as buildpacks

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

### Create builder with buildpacks

```plain
pack create-builder starkandwayne/fizzbuzz-builder -b builder.toml
```

Test the various fixture apps on the builder:

```plain
pack build playtime --builder starkandwayne/fizzbuzz-builder --no-pull --path fixtures/fifteen
pack build playtime --builder starkandwayne/fizzbuzz-builder --no-pull --path fixtures/five
pack build playtime --builder starkandwayne/fizzbuzz-builder --no-pull --path fixtures/three
pack build playtime --builder starkandwayne/fizzbuzz-builder --no-pull --path fixtures/one
```