# [npm](https://github.com/npm/cli) Transitive OS-Specific Shrinkwrap Dependency Error

This repository demonstrates a bug in the [`npm ci`](https://docs.npmjs.com/cli/v10/commands/npm-ci) command which
produces an error when running the command in a package that publishes a [shrinkwrap](https://docs.npmjs.com/cli/v10/commands/npm-shrinkwrap),
and that has a transitive, [_optional_](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#optionaldependencies)
dependency with an [OS constraint](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#os).

## Project Contents

* [`lib1.shrinkwrap`](./lib1.shrinkwrap) - This is a package [published](https://www.npmjs.com/package/@restjohn/issues.transitive-optional-dep.lib1.shrinkwrap)
  with a shrinkwrap file, with a transitive, OS-specific dependency. The dependency, in this case, is
  [`mocha`](https://npmjs.com/package/mocha), which depends on [`chokidar`](https://npmjs.com/package/chokidar),
  which _optionally_ depends on [`fsevents`](https://npmjs.com/package/mocha), which has a
  [constraint](https://github.com/fsevents/fsevents/blob/2db891e51aa0f2975c5eaaf6aa30f13d720a830a/package.json#L7) to
  the `darwin` (macOS) platform.
* [`lib1.lock`](./lib1.lock) - This is essentially the the same package as above, but using a `package-lock.json`,
  [published](https://www.npmjs.com/package/@restjohn/issues.transitive-optional-dep.lib1.lock) without a shrinkwrap
  file.
* [`app1`](./app1) - This is a root package that depends on `lib1.shrinkwrap`.
* [`app2`](./app2) - This is a root package that depends on `lib1.lock`.

## Reproducing the Bug
The bug was reproduced using npm 10.8.1, the latest at the time of this writing.

If you're on a Linux or Windows platform, simply descend into the `app1` directory and run `npm ci`.  This should
produce an error like
```shell
app1 % npm ci
npm error code EBADPLATFORM
npm error notsup Unsupported platform for fsevents@2.3.3: wanted {"os":"darwin"} (current: {"os":"linux"})
npm error notsup Valid os:  darwin
npm error notsup Actual os: linux

npm error A complete log of this run can be found in: /root/.npm/_logs/2024-07-02T13_12_08_752Z-debug-0.log
```

If you're on macOS, you can use a [node docker image](https://hub.docker.com/layers/library/node/20-slim/images/sha256-d71e0d460434ef500c6d962bcdc6ebe717e5939025dd8aa9746dd76fa12ee6e3?context=explore)
something like the following, from the root of this repository.
```shell
docker run -it --entrypoint /bin/bash -v $(pwd):/shrinkwrap_optional_dep node:20-slim
cd /shrinkwrap_optional_dep/app1
npm ci
```

## Other Notes

### Without Shrinkwrap
Running `npm ci` in the `app2` package produces no error, because `app2` depends on the published [`lib1.lock`](https://www.npmjs.com/package/@restjohn/issues.transitive-optional-dep.lib1.lock)
package, which does not use `npm-shrinkwrap.json` as `lib1.shrinkwrap` does.

### `npm install` Works
Running `npm install` in the `app1` package on a non-Darwin platform produces no error.

### Package Tarball Works
Installing `lib1.shrinkwrap` from a package tarball produces no error.  You can test this from your macOS platform
easiest by using the Docker method described above.  From the root repository directory in a macOS shell, execute the
following commands.
```shell
npm pack ./lib1.shrinkwrap
cd app1
npm i ../restjohn-issues.transitive-optional-dep.lib1.shrinkwrap-1.0.0.tgz
cd ..
```
Then, use a Docker container to test `npm ci`.
```shell
docker run -it --entrypoint /bin/bash -v $(pwd):/working node:20-slim
cd /working/app1
npm ci
```

### Remote Package Tarball Works
Installing `lib1.shrinkwrap` in `app1` as a remote tarball produces no error from `npm ci`.  You can test this easily
by using the [`http-server`](https://npmjs.com/package/http-server) package.  From the repository root directory in
your macOS shell, execute the following commands.
```shell
npm pack ./lib1.shrinkwrap
npm i -g http-server
http-server -a 127.0.0.1 .
```
`http-server` is now running in the foreground in that shell.  Open another shell window and execute the following
commands from the root repository directory.
```shell
cd app1
npm i http://127.0.0.1:8080/restjohn-issues.transitive-optional-dep.lib1.shrinkwrap-1.0.0.tgz
cd ..
docker run -it --entrypoint /bin/bash -v $(pwd):/working node:20-slim
cd /working/app1
npm ci
```

### Verdaccio Registry Works
