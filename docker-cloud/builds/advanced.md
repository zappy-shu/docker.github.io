---
description: Automated builds
keywords: automated, build, images
title: Advanced options for Autobuild and Autotest
---

The following options allow you to customize your automated build and automated test processes.

## Environment variables for building and testing

Several utility environment variables are set by the build process, and are
available during automated builds, automated tests, and while executing
hooks.

> **Note**: These environment variables are only available to the build and test
processes and do not affect your service's run environment.

* `SOURCE_BRANCH`: the name of the branch or the tag that is currently being tested.
* `SOURCE_COMMIT`: the SHA1 hash of the commit being tested.
* `COMMIT_MSG`: the message from the commit being tested and built.
* `DOCKER_REPO`: the name of the Docker repository being built.
* `CACHE_TAG`: the Docker repository tag being built.
* `IMAGE_NAME`: the name and tag of the Docker repository being built. (This variable is a combination of `DOCKER_REPO`:`CACHE_TAG`.)

If you are using these build environment variables in a
`docker-compose.test.yml` file for automated testing, declare them in your `sut`
service's environment as shown below.

```none
sut:
  build: .
  command: run_tests.sh
  environment:
    - SOURCE_BRANCH
```


## Override build, test or push commands

Docker Cloud allows you to override and customize the `build`, `test` and `push`
commands during automated build and test processes using hooks. For example, you
might use a build hook to set build arguments used only during the build
process. (You can also set up [custom build phase hooks](#custom-build-phase-hooks) to perform actions in between these commands.)

**Use these hooks with caution.** The contents of these hook files replace the
basic `docker` commands, so you must include a similar build, test or push
command in the hook or your automated process does not complete.

To override these phases, create a folder called `hooks` in your source code
repository at the same directory level as your Dockerfile. Create a file called
`hooks/build`, `hooks/test`, or `hooks/push` and include commands that the
builder process can execute, such as `docker` and `bash` commands (prefixed appropriately with `#!/bin/bash`).

## Custom build phase hooks

You can run custom commands between phases of the build process by creating
hooks. Hooks allow you to provide extra instructions to the autobuild and
autotest processes.

Create a folder called `hooks` in your source code repository at the same
directory level as your Dockerfile. Place files that define the hooks in that
folder. Hook files can include both `docker` commands, and `bash` commands as long as they are prefixed appropriately with `#!/bin/bash`. The builder executes the commands in the files before and after each step.

The following hooks are available:

* `hooks/post_checkout`
* `hooks/pre_build`
* `hooks/post_build`
* `hooks/pre_test`
* `hooks/post_test`
* `hooks/pre_push` (only used when executing a build rule or [automated build](automated-build.md) )
* `hooks/post_push` (only used when executing a build rule or [automated build](automated-build.md) )

### Build hook examples

#### Override the "build" phase to set variables

Docker Cloud allows you to define build environment variables either in the hook files, or from the automated build UI (which you can then reference in hooks).

In the following example, we define a build hook that uses `docker build` arguments to set the variable `CUSTOM` based on the value of variable we defined using the Docker Cloud build settings. `$IMAGE_NAME` is a variable that we provide with the name of the image being built.

```none
docker build --build-arg CUSTOM=$VAR -t $IMAGE_NAME .
```

> **Caution**: A `hooks/build` file overrides the basic [docker build](/engine/reference/commandline/build.md) command
used by the builder, so you must include a similar build command in the hook or
the automated build fails.

To learn more about Docker build-time variables, see the [docker build documentation](/engine/reference/commandline/build/#set-build-time-variables-build-arg).

#### Two-phase build

If your build process requires a component that is not a dependency for your application, you can use a pre-build hook (refers to the `hooks/pre_build` file) to collect and compile required components. In the example below, the hook uses a Docker container to compile a Golang binary that is required before the build.

```bash
#!/bin/bash
echo "=> Building the binary"
docker run --privileged \
  -v $(pwd):/src \
  -v /var/run/docker.sock:/var/run/docker.sock \
  centurylink/golang-builder
```

#### Push to multiple repos

By default the build process pushes the image only to the repository where the build settings are configured. If you need to push the same image to multiple repositories, you can set up a `post_push` hook to add additional tags and push to more repositories.

```none
docker tag $IMAGE_NAME $DOCKER_REPO:$SOURCE_COMMIT
docker push $DOCKER_REPO:$SOURCE_COMMIT
```

## Source Repository / Branch Clones

When Docker Cloud pulls a branch from a source code repository, it performs
a shallow clone (only the tip of the specified branch).  This has the advantage
of minimizing the amount of data transfer necessary from the repository and
speeding up the build because it pulls only the minimal code necessary.

Because of this, if you need to perform a custom action that relies on a different
branch (such as a `post_push` hook), you can't checkout that branch, unless
you do one of the following:

* You can get a shallow checkout of the target branch by doing the following:

		git fetch origin branch:mytargetbranch --depth 1

* You can also "unshallow" the clone, which fetches the whole Git history (and potentially
takes a long time / moves a lot of data) by using the `--unshallow` flag on the fetch:

		git fetch --unshallow origin