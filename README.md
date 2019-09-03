# FizzBuzz as buildpacks

## Simple demo

```plain
mkdir -p ~/workspace/fizzbuzz-app
cd ~/workspace/fizzbuzz-app

echo 3 > Count
pack build fizzbuzz-app --builder starkandwayne/fizzbuzz-builder

echo 5 > Count
pack build fizzbuzz-app --builder starkandwayne/fizzbuzz-builder

echo 15 > Count
pack build fizzbuzz-app --builder starkandwayne/fizzbuzz-builder
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
