---
title: "Travis-CI, ccache, and Object File Compression"
date: 2020-05-15
categories:
  - programming
tags:
  - travis
  - ci
  - ccache
  - LunarG
  - Khronos
  - Vulkan
---

## The GitHub Repository and Travis-CI
At [LunarG](https://www.lunarg.com), we set up [Travis-CI](https://travis-ci.org) to do CI work on submissions to one of the Vulkan GitHub projects, [Vulkan-ValidationLayers](https://github.com/KhronosGroup/Vulkan-ValidationLayers).
This CI [build](https://travis-ci.org/github/KhronosGroup/Vulkan-ValidationLayers) basically does compile checking of pull requests (PRs) and runs some tests.

When we first set it up, we wanted to have the CI builds run as fast as possible.
We wanted the results to be available to PR submitters as soon as possible so that they can get feedback quickly and resubmit their PR if needed.
We also wanted to reduce the load on Travis-CI servers, just to be good Travis citizens.

### Dependent Libraries
One characteristic of this build is that it needs to build some dependent libraries that don't frequently change between builds of the primary (Vulkan-ValidationLayers) repository.
In particular, the [glslang](https://github.com/KhronosGroup/glslang) and [spirv-tools](https://github.com/KhronosGroup/SPIRV-Tools) repositories are built every time the CI build runs.
These two repositories take a significant amount of time to build and contribute to a good portion of the CI build time.

However, we "lock-in" specific versions of these repositories via a "known-good" mechanism and these versions change only every few weeks or so.
That means that the CI build is building the same sources several times a day, starting with an empty or "clean" build tree each time.
This seems like a terrible inefficiency.

This sort of workflow is similar to a developer's workflow where they clean their build frequently, possibly to make sure that there aren't any problems with build dependencies, etc.
This is where ccache becomes very useful.

## ccache
Briefly, [ccache](https://ccache.dev) is a compiler wrapper that collects all the input to a compiler and creates a hash.
This hash includes the source code, include files, compiler options, compiler version, and anything else that can affect the contents of an object file.
It then uses the hash to look up the object file in a "database" cache and if found, produces the object file as the compiler output instead of compiling the file.
If the hash is not found, then it compiles the file normally and stores the object file in the database using the hash.

Travis-CI supports ccache and we can enable it with a setting in its [config file](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/master/.travis.yml).
In order to be useful, the ccache cache must persist across CI builds.
For a normal developer, the ccache cache resides someplace on their machine.
But Travis-CI starts with a clean VM, effectively, for each build and so there can be no persistent cache.
So, the Travis-CI support for ccache involves storing the ccache cache "in the cloud" someplace.
When ccache is enabled, Travis-CI fetches the cache from the cloud before the build starts and stores it back in the cloud at the end of the build.
The cache size is limited to 500MB, which is pretty respectable, considering this is a free service.
Each "matrix" job in a CI build gets its own cache and each branch gets its own cache.
This can explain the reason for the somewhat small cache size limit.
There can be a lot of branches and jobs with caches attached to them.

And something that is really cool is that Travis will combine caches from derivative branches.
For example, if I make a PR, the CI build will be building my PR branch for the first time and I would expect an empty ccache cache and no build speed improvement since I would have to build everything for the first time.
Instead of starting with an empty ccache cache, Travis-CI fetches the cache from the cache used to build the master branch at the point where the PR branch was made.

## Performance
When we first set all this up a couple of years ago, we were getting decent performance from the cache.
We set up the Travis config to reset ccache stats before the job runs and then display the stats at the end of the job.
We were getting the expected cache hit rate and the Travis builds were completed faster than they would have without ccache enabled.

But at some point, the cache hit rate dropped to zero and the Travis builds were taking a long time again.
I don't know exactly when or why this happened, but my best guess is that Travis turned on the 500MB cache size limit and that killed us.

### Cache Thrash
The first look at the problem showed that the cache size was close to the 500MB max.
The first attempt to address this involved reducing what went into the cache.
I tried turning ccache off for some of the components in the build and building things without debug info.
The hit rate went up, but not by enough and these tweaks were not acceptable configurations.

I then built the repo locally on my machine with ccache enabled so I could more easily examine the cache behavior.
I was free of the 500MB limit on my machine, so I could see the size of the ccache at each point in the build.
I quickly discovered that the glslang/spirv-tools build took up over 1GB of cache.
So that explains it.
The glslang/spriv-tools build filled the cache.
Then, the "rest of the build" pushed the glslang/spirv-tools objects out of the cache, replacing them with their own.
When the next CI build runs, the glslang/spirv-tools objects were not found in the cache, so they are rebuilt and pushed into the cache, continuing the cycle.

## The Solution
I was sort of stuck at this point.
I could change the build to cache only a subset of the components and build release instead of debug, but that's changing something that I didn't want to change and I did want to cache everything.
I also wasn't certain if we could "buy" more Travis cache space even if we wanted to.

By total coincidence, I hit a problem on another project where we were concerned about the size of a static library archive file (.a file) that needed to be distributed to a large number of developers.
I never thought that object files (and their .a containers) were that compressible since I assumed they contain mostly binary data.
We ended up compressing the library into a tar.zx file and found that it could get down to about 10X smaller.
I suppose that object files contain symbol strings that are made up of a small subset of the encoding space.
That is, the ASCII character codes are a pretty small subset of the 256 possible values.
This makes at least part of an object file compress as well as text does.
Perhaps a lot of C++ code increases the amount of text as well, with the long symbol names.
There may also be long runs of zeros that compress really well.
All of these things facilitate compression.
Live and learn.

I dug into the ccache docs and was pleasantly surprised to find a compression option.
I was surprised by this since, again, I didn't think object files were ever worth compressing.
But the ccache authors apparently knew better than I.

So, I [turned on](https://github.com/KhronosGroup/Vulkan-ValidationLayers/commit/54169afb2789a095b66240df1b3006f1a2d3ac62) ccache compression in the travis config file, and observed 100% cache hit rates (for builds that had no source changes between them).
The wall-clock time for a CI build improved from about 20-30 minutes to 5-10 minutes.
There's a good deal of variance in these numbers due to Travis-CI system load.
A build can also run a bit longer if there was a source code change that forces recompiles.

Easy!

## Other Solutions
There are a couple of other approaches that I considered and are worth mentioning here.
The ccache solution is nice because it is fully automatic and needs no maintenance or attention.
But we're pretty lucky that it worked out.
For other situations, something else may be needed.

### Install Binaries
If we had a source for the right pre-built versions of the dependent libraries, the Travis job could fetch and install them instead.

### Docker Containers
Our Travis build starts out by installing a rather large set of build prerequisites.
For example, we grab a specific version of CMake.
And there are some packages installed with `apt-get`.

We could build up a Docker image that has all of these dependencies pre-installed.
We could also go a step further and install the dependencies that are more specific to this project such as the glslang and spirv-tools libraries.
Once Travis spools up this Docker image as a container in a build, there would be near-zero additional setup time, resulting in a pretty optimum build.

The downside is that the Docker image would need maintenance.
If we did put in glslang and spirv-tools, we'd have to update the image whenever we bumped the version of those libraries and republish the image, probably to someplace like DockerHub.

But a possible upside is that this same Docker image could be used by developers to work on Vulkan apps or ValidationLayers.
We'd have to work out details such as accessing the GPU from the container and any other Docker considerations for this to be useful.

In the end, I doubt that the Travis build performance would be significantly better in the Docker container compared to ccache.
It probably isn't worth the trouble for a couple of minutes of CI time.