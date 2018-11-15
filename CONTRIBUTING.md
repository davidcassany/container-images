# Contributing

Welcome to the kubic-project! If you are interested in contributing to the [kubic container-images repo](README.md) then continue reading this document as it contains information on how the [kubic container images](https://registry.opensuse.org/cgi-bin/cooverview) is populated and other information that is pertinent to building kubic containers.

## Contributing by modifying an existing image

There are two ways to do it:

### 1 Modify the kiwi file (at GitHub)

The sources for our openSUSE container images used in Kubic, can be found at the [kubic container-images repo](github: https://github.com/kubic-project/container-images). If you just want to modify an already existing image, then you don't have to do anything special. As soon as your PR gets approved and merged, then the changes are committed to the OBS. This triggering happens thanks to our [internal ConcourseCI](http://concourse.caasp.suse.net/teams/main/pipelines/imaging-head) pipeline. This CI system is not meant to do any testing. It just connects Github with the appropriate devel projects in build service and triggers a new build of the images into their _devel_ projects. If the build is _green_ then your image gets released into the [openSUSE container registry](https://registry.opensuse.org/cgi-bin/cooverview).

> This is the reason why sometimes in OBS you will encounter the following commit message posted by our ConcourseCI: "Containers Team (containersteam) committed 27 days ago (revision 11) - Updated from concourse CI"

__NOTE__: For the people who are not connected within SUSE RnD network, you cannot have access to ConcourseCI -- actually you don't have to. All you need to know is that as soon as you commit something in this repository, then OBS will be triggered.

### 2 Bump the package version

To get a new image accepted into our [openSUSE container registry](https://registry.opensuse.org/cgi-bin/cooverview), it is required that all the RPM packages related to that container image, to be present in [openSUSE Factory project](https://build.opensuse.org/project/show/openSUSE:Factory). For example, for building the _kubic-dex image_, first we had to build the [caasp-dex](https://build.opensuse.org/package/show/openSUSE:Factory/caasp-dex) RPM package that provides the binary that is used as a part of the entrypoint for this image. As soon as you modify a package that is included in the _.kiwi_ definition of a container image, then it will automatically schedule a new build. In other words, if you ever modify the _RPM pkg_ that is related to an image, then OBS will automatically trigger a new rebuild of your image and if it's _green_ it will publish it to the registry.

## Contributing by adding an image:

In case you would like to build a new container image that does not exist yet in our registry, then you have to send a _Merge Request_ to the [kubic automation repo](https://github.com/kubic-project/automation) in order to activate add the instructions for the Concourse pipeline to take this new image of yours into account. Otherwise, you have to manually create a _SubmitRequest_ to the appropriate _devel project_ in OBS. To highlight this for the reviewer, a proposed label for this should be `new_image`.

## Test before you submit

### Fetch the code from OBS

Nowhere in this process there is any testing automation. Hopefully, since you are building and modifying these images, you should be able to test your own changes by running some local tests. To do that, you can build your container image locally using [kiwi image building system](https://doc.opensuse.org/projects/kiwi/doc/). To do that you have to perform a checkout of the image you want from OBS. Then inside that project/branch you have, you can just modify the _.kiwi_ file accordingly. When your image is building in the OBS, it uses [pre_checking.sh script](https://github.com/kubic-project/container-images/blob/master/pre_checkin.sh), but you __should not use it__ when you are testing locally. This is because the project in OBS is templated in a way that certain _placeholders_ are replaced by values taken from GitHub -- that means that it will accidentally overwrite your local changes.

> __WARNING__: You should NOT use this `pre_checking.sh` for testing locally

### Templated structure

As soon as you have _cloned/checked-out_ the code from OBS to your local dis, then it's time to build the container image. But before doing that it would be wise to be informed first about an existing a bug in the `osc` tool that does not trigger correctly the `replace_using_package_version` service. This is an OBS service that ensures that the image we are building has the same tag with the value of the OBS package version that provides. To understand this better, let's look for example at the [kubic-caasp-dex-image](https://build.opensuse.org/package/show/devel:CaaSP:kubic-container/kubic-caasp-dex-image).

```bash
_service # Synchronisation between the tag of the container image with the RPG pkg version
_servicedata # Obsolete. Used to autogenerate the .changes with the obs_scm service, but it was not good enough
caasp-caasp-dex-image.changes
kubic-caasp-dex-image.changes
caasp-caasp-dex-image.kiwi # Autogenerated. Intended for IBS usage (SUSE CaaSP)
kubic-caasp-dex-image.kiwi # Autogenerated. Intended for OBS usage (openSUSE Kubic)
caasp-dex-image.kiwi.ini # Template file from which the above .kiwi files are generated
pre_checkin.sh # Do not use that when building locally
```

### Read the kiwi file

If you are curious enough, you can read the [kubic-caasp-dex-image.kiwi](https://build.opensuse.org/package/view_file/devel:CaaSP:kubic-container/kubic-caasp-dex-image/kubic-caasp-dex-image.kiwi?expand=1) file. You need to pay attention to the `name="kubic-caasp-dex">` -- this gets generated from the template, parsing the [_PRODUCT_](https://github.com/kubic-project/container-images/blob/master/caasp-dex-image/caasp-dex-image.kiwi.ini#L5) placeholder and it refers to the name of the package within OBS (_as artifact of the devel project -- it should not be confused with the RPM package in any case_). For those who are not familiar with _kiwi image definitions_, the _kiwi_ file is similar to the _spec_ file for RPM, or _Dockerfile_ for docker containers. They are instructions on how to build/bake a container image through OBS. This duplication of kiwi files you see there, happens because the ConcourseCI is using `copypack`. _Copypac_ is a pure plain copy from one project to another, taking all the sources (as they are) to another project. It doesn't create any link or anything like that. However, SUSE maintainers should not forget that `copypac` is not happening by default, but only if the image already exists in SLE (IBS). This way the automation is only responsible of tracking updates, but not creating new OBS packages, the very first `copypac` to SLE needs to be done manually.

### Read the service file

Continuing our example, the next interestng part is taking a look at the [service](https://build.opensuse.org/package/view_file/openSUSE:Factory/caasp-dex/_service?expand=1) file of the [kubic-caasp-dex-image.kiwi](https://build.opensuse.org/package/view_file/devel:CaaSP:kubic-container/kubic-caasp-dex-image/kubic-caasp-dex-image.kiwi?expand=1). Here you should focus on the `<service name="replace_using_package_version"` section.

```xml
    <service name="replace_using_package_version" mode="buildtime">
        <param name="file">kubic-caasp-dex-image.kiwi</param>
        <param name="regex">%%SHORT_VERSION%%</param>
        <param name="parse-version">minor</param>
        <param name="package">caasp-dex</param>
    </service>
```

This service is executed during _buildtime_. Its job is to verify the package version for the `%%placeholder%%`. It's looking for the RPM package `<param name="package">caasp-dex</param>` and the regex is just a placeholder for any %%SHORT_VERSION% with the MINOR version (_if you don't know what MINOR version stands for, keep reading the next paragraph_).

### Tags and versions

Based the previous example, issueing `rpm -q caasp-dex` from a TW machine we will get `caasp-dex-2.7.1-1.2.x86_64` -- _this is version is expected to be higher when you read this article_. The versioning can be explained as follows:

```
MAJOR: 2
MINOR: 7
PATCH: 1
```

Tags for images:

```bash
- 2.7        is following the pattern of <name>:<short_version>
- 2.7.1      is following the pattern of <name>:<long_version>
- 2.7.1-9.19 is following the patter of <name>:<long_version>-<release>
```

You can verify that by listing the available image in our [openSUSE container registry](https://registry.opensuse.org/cgi-bin/cooverview) using a tool called [reg](https://github.com/genuinetools/reg):

```bash
$ reg ls registry.opensuse.org | grep dex
devel/caasp/kubic-container/container/kubic/caasp-dex                                                              2.7, 2.7.1, 2.7.1-9.19
devel/caasp/kubic-container/container_arm/kubic/caasp-dex                                                          2.7, 2.7.1, 2.7.1-9.45
```

The `%%SHORT_VERSION%%` placeholder it is used to keep aligned the tag and the RPM package version. So this `replace_using_package_version` service grabs the version you want and makes sure that your container image and the RPM package withing TW have the same tag/version. How it does that? Well, first it has to find the package (it needs to be inside the image obviously, so it has to be already submitted to Factory). From this RPM pkg, it takes the version using rpm commands, and then it uses it as TAG for the container image.

### Workaround for the bug

However, this `replace_using_package_version` functionality doesn't happen when you build locally at your computer -- _unless_ you follow this workaround (__recommended__): In order for this _service_ to be usable at build time, it needs to be included into the build environment configuration. For some strange reason the OBS solver includes that service as a build requirement, but it does not pre-install it at the local builds. So you have to use the "pre-installed" call. This is:

```
# preinstall: obs-service-$nameoftheservice
preinstall: obs-service-replace_using_package_version
```

This has to be included in the project configuration file: `osc meta prjconf -e`.  Again, this is something you need to include this line to build it locally.


### Build and test

Finally, to build it, you can use the typical _build_ command by specifying the repository we want our image to be published. In this special case, this repository is "container".: `osc build <repo>`.

## Example

A summary of all the previous steps, using again the [kubic-caasp-dex-image](https://build.opensuse.org/package/show/devel:CaaSP:kubic-container/kubic-caasp-dex-image) as an example.

> Source: devel:CaaSP:kubic-container / kubic-caasp-dex-image

> Devel Project: https://build.opensuse.org/project/show/devel:CaaSP:kubic-container
                 https://build.opensuse.org/package/show/devel:CaaSP:kubic-container/kubic-caasp-dex-image

In other words: `osc branchco <project name> <package name>`

Issuing the command `osc branchco devel:CaaSP:kubic-container kubic-caasp-dex-image` will create the following branch:
 * Branch Project name: home:YOURNAME:branches:devel:CaaSP:kubic-container
 * Branch Package name: kubic-caasp-dex-image

```bash
tux@ultron:~/obs> osc branchco devel:CaaSP:kubic-container kubic-caasp-dex-image
A    home:YOURNAME:branches:devel:CaaSP:kubic-container
A    home:YOURNAME:branches:devel:CaaSP:kubic-container/kubic-caasp-dex-image
A    home:YOURNAME:branches:devel:CaaSP:kubic-container/kubic-caasp-dex-image/_service
A    home:YOURNAME:branches:devel:CaaSP:kubic-container/kubic-caasp-dex-image/_servicedata
A    home:YOURNAME:branches:devel:CaaSP:kubic-container/kubic-caasp-dex-image/caasp-caasp-dex-image.changes
A    home:YOURNAME:branches:devel:CaaSP:kubic-container/kubic-caasp-dex-image/caasp-caasp-dex-image.kiwi
A    home:YOURNAME:branches:devel:CaaSP:kubic-container/kubic-caasp-dex-image/caasp-dex-image.kiwi.ini
A    home:YOURNAME:branches:devel:CaaSP:kubic-container/kubic-caasp-dex-image/kubic-caasp-dex-image.changes
A    home:YOURNAME:branches:devel:CaaSP:kubic-container/kubic-caasp-dex-image/kubic-caasp-dex-image.kiwi
A    home:YOURNAME:branches:devel:CaaSP:kubic-container/kubic-caasp-dex-image/pre_checkin.sh
At revision 282f53e814fca88064c241a629e3185a.
Note: You can use "osc delete" or "osc submitpac" when done.
```

Any personal branch can be found at the subprojects of your homeproject. e.g. browsing at your the subprojects `https://build.opensuse.org/project/subprojects/home:YOURNAME` by modyfing the URI.

> Subproject: branches:devel:CaaSP:kubic-container
> Title: Branch project for package kubic-caasp-dex-image

URL: `https://build.opensuse.org/project/show/home:YOURNAME:branches:devel:CaaSP:kubic-container`

Now go into that:

```bash
tux@ultron:~/obs> cd home\:YOURNAME\:branches\:devel\:CaaSP\:kubic-container/kubic-caasp-dex-image
```

Modify the configuration of the project to execute the service `replace_using_package_version` during buildtime. To do that, type: `osc meta prjconf -e` and copypaste this line:

```bash
preinstall: obs-service-replace_using_package_version
```

Now build locally by typing: `sudo osc build container`

Output when it finishes:

```
/var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/kubic-caasp-dex.x86_64-4.0.1-Build.packages
/var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/kubic-caasp-dex.x86_64-4.0.1-Build.verified
/var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/kubic-caasp-dex.x86_64-4.0.1.metadata
/var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/kubic-caasp-dex.x86_64-4.0.1.tag
/var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/image.changes
/var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/kubic-caasp-dex.x86_64-4.0.1-Build.docker.tar
/var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/kubic-caasp-dex.x86_64-4.0.1-Build.docker.tar.sha256
/var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/kubic-caasp-dex.x86_64-4.0.1-Build.docker.containerinfo
/var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/kubic-caasp-dex.x86_64-4.0.1-Build.basepackages
```

Load the image into docker:

```bash
$ docker load -i /var/tmp/build-root/container-x86_64/usr/src/packages/KIWI/kubic-caasp-dex.x86_64-4.0.1-Build.docker.tar
638ff2536f74: Loading layer [==================================================>]  34.51MB/34.51MB
Loaded image: kubic/caasp-dex:2.7
```

Verify that is loaded:

```
$ docker image ls kubic/caasp-dex
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
kubic/caasp-dex     2.7                 dca2f32bc6c4        20 hours ago        148MB
```

Then, you should be able to know how to use this package and test it.
