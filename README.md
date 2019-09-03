# FizzBuzz as buildpacks

## Test buildpacks without builder

```plain
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
