---
categories:
- Naked Blogging
- iOS Development
date: 2021-01-17T22:45:00-07:00
description: In this post, I will show you how to build libgit2 for use in iOS and Ctalyst applications and package it up using Swift Package Manager to include in your own applications.
draft: false
resources:
- name: "featured-image"
  src: "feature.jpg"
- name: "featured-image-preview"
  src: "feature_small.jpg"
tags:
- Catalyst
- CMake
- Git
- iOS
- libgit2
- Swift Package Manager
title: "Build libgit2 for iOS and Catalyst"
---
Using open source libraries can be a blessing or a curse. They can be a blessing in the short term if they help you to accomplish an immediate task. They can be a curse if they're not maintained. In this article, I will take you through my journey to consume libgit2 and why I did it.

<!--more-->

There's no denying the impact that [Git](https://git-scm.com) has had on software development. It's all over the place and its distributed version control model has made software development a lot more productive and fun. I like being able to make small commits in my local repository to persist my state and later be able to easily share my code when it's fuly ready to be shared. Another feature of Git that I enjoy is that the core of Git is implemented in a reusable open source library named [libgit2](https://libgit2.org).

I can think of several good scenarios for using libgit2 in an application. Specifically for my [Naked Blogging]({{< ref introducing-naked-blogging >}}) app, I can use libgit2 to clone my website repository from GitHub, commit my changes locally, share them between devices, and when I am ready, publish my new content back to my GitHub repository to be published on my blog and ready by you. libgit2 already has a [Swift language binding](https://github.com/SwiftGit2/SwiftGit2), but at the time of this writing it was last updated in May 2019 and the latest version of libgit2 is 1.1.0 that was released on October 12, 2020. There is also [Objective-Git](https://github.com/libgit2/objective-git), but it was last updated on October 27, 2018. Also, I'm not exactly sure how they build their frameworks and I have specific desires. I want the C library that will work for both iOS on ARM, iOS Simulator, and Catalyst on macOS (targeting Intel chips; I don't have access to an M1, but I believe the iOS framework would work).  I also want to share and consume these frameworks using Swift Package Manager. So instead of consuming the older frameworks, I'm going to pave forward on my own and create my own libgit2-based framework.

To build and use libgit2, I need to actually build and redistribute three libraries or frameworks:

* [libgit2](https://github.com/libgit2/libgit2)
* [OpenSSL](https://github.com/openssl/openssl)
* [libssh2](https://github.com/libssh2/libssh2)

Since I am building the libraries myself, I'm going to target the latest released versions to make sure that I have all of the features and defect fixes in each of those libraries.

## OpenSSL

OpenSSL is an open source project that provides a cryptographic algorithm library and helps to support the Transport Layer Security protocols for TCP and HTTP network connections. I haven't looked into the libgit2 code to see exactly how it is being used, but I'd guess it's being used for the HTTPS protocol support (although you know what they say about assumptions...). If I run into any compatibility issues with libgit2 or libssh2, I will backtrack to an earlier version.

At the time of this writing, the latest version of OpenSSL is [1.1.1i](https://github.com/openssl/openssl/releases/tag/OpenSSL_1_1_1i), so I'll start there. I will start by creating a new Git repository for my libgit2 framework, which I will call `libgit2-ios`. I will next add the source code for OpenSSL 1.1.1i to my repository as a Git submodule. I like to put submodules in an `External` directory, so OpenSSL will be located at `External/openssl`.

    git submodule add https://github.com/openssl/openssl.git External/openssl
    cd External/openssl
    git checkout OpenSSL_1_1_1i

The first command will create the submodule and will checkout the `master` branch for OpenSSL. The second and third commands navigate me into the OpenSSL submodule and checks out the `OpenSSL_1_1_1i` tag which gives me the version of the source code included in the OpenSSL 1.1.1i release.

OpenSSL has its own build system using Make, but to successfully build OpenSSL for use in iOS applications, we need to make a modification. Specifically, for iOS redistribution, we want to enable [bitcode](https://developer.apple.com/documentation/xcode/reducing_your_app_s_size/doing_basic_optimization_to_reduce_your_app_s_size) to reduce the size of the application when exporting an archive for distribution or uploading to App Store Connect. I will extend the OpenSSL build system to add support for bitcode. To do so, I will create new configurations for the target platforms that I am interested in and will store them in a new configuration file that I will reference when I build OpenSSL. I placed this file in `External/openssl-config/ios-and-catalyst.conf`:

{{< highlight plain "linenos=table" >}}
my %targets = (
    "openssl-ios" => {
        inherit_from => [ "ios-xcrun" ],
        cflags => add("-fembed-bitcode"),
    },
    "openssl-ios64" => {
        inherit_from => [ "ios64-xcrun" ],
        cflags => add("-fembed-bitcode"),
    },
    "openssl-iossimulator" => {
        inherit_From => [ "iossimulator-xcrun" ],
    },
    "openssl-catalyst" => {
        inherit_From => ["darwin64-x86_64-cc" ],
        cflags => add("-target x86_64-apple-ios-macabi")
    },
)
{{< /highlight >}}

The next thing that I did was to create a Bash shell script to automate building OpenSSL and producing the [XCFrameworks](https://developer.apple.com/documentation/swift_packages/distributing_binary_frameworks_as_swift_packages) for redistribution and linking:

{{< admonition type="note" title="About the Shell Script..." open="true" >}}
I created this shell script quite some time ago. I borrowed the format for it from another GitHub repository that I came across, and unfortunately I lost track of that repository. If my code looks familiar to yours, please contact me and I'll update the blog post to attribute you as the inspiration for my script.
{{< /admonition >}}

{{< highlight bash "linenos=table" >}}
SCRIPT_DIR=$(dirname $0)
pushd $SCRIPT_DIR/.. > /dev/null
ROOT_PATH=$PWD
popd > /dev/null

CONFIGURATIONS="ios ios64 iossimulator catalyst"
for CONFIGURATION in $CONFIGURATIONS
do
    echo "Building OpenSSL for $CONFIGURATION"

    rm -rf /tmp/openssl
    cp -r External/openssl /tmp

    pushd /tmp/openssl > /dev/null

    LOG="/tmp/openssl-$CONFIGURATION.log"
    rm -f $LOG

    OUTPUT_PATH=$ROOT_PATH/build/openssl/$CONFIGURATION
    rm -rf $OUTPUT_PATH
    mkdir -p $OUTPUT_PATH

    ./Configure "openssl-$CONFIGURATION" --config=$ROOT_PATH/External/openssl-config/ios-and-catalyst.conf --prefix=$OUTPUT_PATH >> $LOG 2>&1
    make >> $LOG 2>&1
    make install >> $LOG 2>&1

    popd > /dev/null
done

echo "Creating the universal library for iOS"

OUTPUT_PATH=$ROOT_PATH/build/openssl/lib
rm -rf $OUTPUT_PATH
mkdir -p $OUTPUT_PATH
lipo -create \
    $ROOT_PATH/build/openssl/ios/lib/libcrypto.a \
    $ROOT_PATH/build/openssl/ios64/lib/libcrypto.a \
    -output $OUTPUT_PATH/libcrypto.a
lipo -create \
    $ROOT_PATH/build/openssl/ios/lib/libssl.a \
    $ROOT_PATH/build/openssl/ios64/lib/libssl.a \
    -output $OUTPUT_PATH/libssl.a

echo "Creating the OpenSSL XCFrameworks"

LIB_PATH=$ROOT_PATH/lib
LIBCRYPTO_PATH=$LIB_PATH/libcrypto/libcrypto.xcframework
LIBSSL_PATH=$LIB_PATH/libssl/libssl.xcframework
rm -rf $LIBCRYPTO_PATH
rm -rf $LIBSSL_PATH
mkdir -p $LIB_PATH

xcodebuild -create-xcframework \
    -library $ROOT_PATH/build/openssl/lib/libcrypto.a \
    -library $ROOT_PATH/build/openssl/iossimulator/lib/libcrypto.a \
    -library $ROOT_PATH/build/openssl/catalyst/lib/libcrypto.a \
    -output $LIBCRYPTO_PATH

xcodebuild -create-xcframework \
    -library $ROOT_PATH/build/openssl/lib/libssl.a \
    -headers $ROOT_PATH/build/openssl/ios/include \
    -library $ROOT_PATH/build/openssl/iossimulator/lib/libssl.a \
    -headers $ROOT_PATH/build/openssl/iossimulator/include \
    -library $ROOT_PATH/build/openssl/catalyst/lib/libssl.a \
    -headers $ROOT_PATH/build/openssl/catalyst/include \
    -output $LIBSSL_PATH

pushd $LIB_PATH/libcrypto > /dev/null
zip -r ../libcrypto.zip .
popd > /dev/null

pushd $LIB_PATH/libssl > /dev/null
zip -r ../libssl.zip .
popd > /dev/null

echo "Done; cleaning up"
rm -rf /tmp/openssl
{{< /highlight >}}

Lines 6 through 28 in the code sample start by using the OpenSSL Make script to build the libcrypto and libssl libraries for iOS (32-bit and 64-bit), iOS Simulator, and macOS Catalyst (Intel).The script will run `make install` to copy the libraries and header files to a temporary location in my repository. I will need the libraries at a later point to build both libssh2 and libgit2. In lines 32 through 42, I am taking the 32-bit and 64-bit iOS libraries and I an using the `lipo` tool to create a *fat* library containing both architectures. Finally in lines 46-74, I am creating the XCFramework bundles containing all of the architectures and then packaging the XCFrameworks into ZIP archives for redistribution using Swift Package Manager.

Now I can build OpenSSL by running the script (I placed my script in `bin/build_openssl.sh`):

    bin/build_openssl.sh

{{< admonition type="tip" title="Building OpenSSL Takes a While" open="true" >}}
OpenSSL is not the speediest framework to build. We're also building it 4 times, once for each supported architecture:

* 32-bit iOS
* 64-bit iOS
* iOS Simulator
* macOS Catalyst

Now might be a good time to go to lunch or go get a drink.
{{< /admonition >}}

Once the build script completes, you should see `lib/libssl.zip` and `lib/libcrypto.zip`. These ZIP archives contain the binary frameworks for OpenSSL packaged as XCFrameworks and can be redistributed.

## libssh2

[libssh2](https://github.com/libssh2/libssh2) is an open source implementation of the [SSH2](https://en.wikipedia.org/wiki/SSH_(Secure_Shell)) protocol. libgit2 uses libssh2 to interact with remote Git servers using the SSH protocol. libgit2 can clone repositories or synchronize changes between repositories over SSH2. At the time of writing, the latest libssh2 version in 1.9.0, so that is what I will use. libssh2 has dependencies on OpenSSL, so I will use the 1.1.1i release that I built previously to satisfy that dependency.

libssh2 uses [CMake](https://cmake.org) for its build system. In order to get libssh2 to build successfully for me on iOS, I had to create a special configuration file for CMake. Fortunately, I found another on the Internet that I could use as a template and I made some minor modifications for supporting Catalyst.

{{< admonition type="note" title="About the CMake Script..." open="true" >}}
Like with OpenSSL, I cannot take credit for the complete script. I am not the original author. And unfortunately, like with OpenSSL, I cannot find the original source to give the author proper credit. Unfortunately, I worked on this months ago and lost all of my notes. If you are the author or have seen similar code, please contact me directly and I'll happily update this article with proper attribution.
{{< /admonition >}}

{{< highlight cmake "linenos=table" >}}
# This file is based off of the Platform/Darwin.cmake and Platform/UnixPaths.cmake
# files which are included with CMake 2.8.4
# It has been altered for iOS development

# Options:
#
# IOS_PLATFORM = OS (default) or SIMULATOR
#   This decides if SDKS will be selected from the iPhoneOS.platform or iPhoneSimulator.platform folders
#   OS - the default, used to build for iPhone and iPad physical devices, which have an arm arch.
#   SIMULATOR - used to build for the Simulator platforms, which have an x86 arch.
#
# CMAKE_IOS_DEVELOPER_ROOT = automatic(default) or /path/to/platform/Developer folder
#   By default this location is automatcially chosen based on the IOS_PLATFORM value above.
#   If set manually, it will override the default location and force the user of a particular Developer Platform
#
# CMAKE_IOS_SDK_ROOT = automatic(default) or /path/to/platform/Developer/SDKs/SDK folder
#   By default this location is automatcially chosen based on the CMAKE_IOS_DEVELOPER_ROOT value.
#   In this case it will always be the most up-to-date SDK found in the CMAKE_IOS_DEVELOPER_ROOT path.
#   If set manually, this will force the use of a specific SDK version
#
# IOS_BITCODE = 1/0: Enable bitcode or not. Only iOS >= 6.0 device build can enable bitcode. Default is enabled.

# Macros:
#
# set_xcode_property (TARGET XCODE_PROPERTY XCODE_VALUE)
#  A convenience macro for setting xcode specific properties on targets
#  example: set_xcode_property (myioslib IPHONEOS_DEPLOYMENT_TARGET "3.1")
#
# find_host_package (PROGRAM ARGS)
#  A macro used to find executable programs on the host system, not within the iOS environment.
#  Thanks to the android-cmake project for providing the command

# Standard settings
set (CMAKE_SYSTEM_NAME Darwin)
set (CMAKE_SYSTEM_VERSION 1)
set(CMAKE_CROSSCOMPILING TRUE)
set (UNIX TRUE)
set (APPLE TRUE)
set (IOS TRUE)

if(NOT DEFINED IOS_BITCODE) # check xcode/clang version? since xcode 7
  set(IOS_BITCODE 1)
endif()
set(IOS_BITCODE_MARKER 0)

# Required as of cmake 2.8.10
set (CMAKE_OSX_DEPLOYMENT_TARGET "" CACHE STRING "Force unset of the deployment target for iOS" FORCE)

# Determine the cmake host system version so we know where to find the iOS SDKs
find_program (CMAKE_UNAME uname /bin /usr/bin /usr/local/bin)
if (CMAKE_UNAME)
	exec_program(uname ARGS -r OUTPUT_VARIABLE CMAKE_HOST_SYSTEM_VERSION)
	string (REGEX REPLACE "^([0-9]+)\\.([0-9]+).*$" "\\1" DARWIN_MAJOR_VERSION "${CMAKE_HOST_SYSTEM_VERSION}")
endif (CMAKE_UNAME)

set(CMAKE_AR ar CACHE FILEPATH "" FORCE)
set(CMAKE_RANLIB ranlib CACHE FILEPATH "" FORCE)

# Skip the platform compiler checks for cross compiling
set (CMAKE_CXX_COMPILER_WORKS TRUE)
set (CMAKE_C_COMPILER_WORKS TRUE)

# All iOS/Darwin specific settings - some may be redundant
set (CMAKE_SHARED_LIBRARY_PREFIX "lib")
set (CMAKE_SHARED_LIBRARY_SUFFIX ".dylib")
set (CMAKE_SHARED_MODULE_PREFIX "lib")
set (CMAKE_SHARED_MODULE_SUFFIX ".so")
set (CMAKE_MODULE_EXISTS 1)
set (CMAKE_DL_LIBS "")

if(IOS_BITCODE)
    set(BITCODE_FLAGS "-fembed-bitcode")
  elseif(IOS_BITCODE_MARKER)
    set(BITCODE_FLAGS "-fembed-bitcode-marker")
  endif()

set (CMAKE_C_OSX_COMPATIBILITY_VERSION_FLAG "-compatibility_version ")
set (CMAKE_C_OSX_CURRENT_VERSION_FLAG "-current_version ")
set (CMAKE_CXX_OSX_COMPATIBILITY_VERSION_FLAG "${CMAKE_C_OSX_COMPATIBILITY_VERSION_FLAG}")
set (CMAKE_CXX_OSX_CURRENT_VERSION_FLAG "${CMAKE_C_OSX_CURRENT_VERSION_FLAG}")

# Setup iOS platform unless specified manually with IOS_PLATFORM
if (NOT DEFINED IOS_PLATFORM)
	set (IOS_PLATFORM "OS")
endif ()
set (IOS_PLATFORM ${IOS_PLATFORM} CACHE STRING "Type of iOS Platform")

# Hidden visibilty is required for cxx on iOS 
if (${IOS_PLATFORM} STREQUAL "CATALYST")
	set (CMAKE_C_FLAGS_INIT "-target x86_64-apple-ios-macabi")
	set (CMAKE_CXX_FLAGS_INIT "-fvisibility=hidden -fvisibility-inlines-hidden -target x86_64-apple-ios-macabi")
else ()
	set (CMAKE_C_FLAGS_INIT "${BITCODE_FLAGS}")
	set (CMAKE_CXX_FLAGS_INIT "-fvisibility=hidden -fvisibility-inlines-hidden ${BITCODE_FLAGS}")
endif ()

set (CMAKE_C_LINK_FLAGS "-Wl,-search_paths_first ${CMAKE_C_LINK_FLAGS}")
set (CMAKE_CXX_LINK_FLAGS "-Wl,-search_paths_first ${CMAKE_CXX_LINK_FLAGS}")

set (CMAKE_PLATFORM_HAS_INSTALLNAME 1)
set (CMAKE_SHARED_LIBRARY_CREATE_C_FLAGS "-dynamiclib -headerpad_max_install_names")
set (CMAKE_SHARED_MODULE_CREATE_C_FLAGS "-bundle -headerpad_max_install_names")
set (CMAKE_SHARED_MODULE_LOADER_C_FLAG "-Wl,-bundle_loader,")
set (CMAKE_SHARED_MODULE_LOADER_CXX_FLAG "-Wl,-bundle_loader,")
set (CMAKE_FIND_LIBRARY_SUFFIXES ".dylib" ".so" ".a")

# hack: if a new cmake (which uses CMAKE_INSTALL_NAME_TOOL) runs on an old build tree
# (where install_name_tool was hardcoded) and where CMAKE_INSTALL_NAME_TOOL isn't in the cache
# and still cmake didn't fail in CMakeFindBinUtils.cmake (because it isn't rerun)
# hardcode CMAKE_INSTALL_NAME_TOOL here to install_name_tool, so it behaves as it did before, Alex
if (NOT DEFINED CMAKE_INSTALL_NAME_TOOL)
	find_program(CMAKE_INSTALL_NAME_TOOL install_name_tool)
endif ()

# Setup building for arm64 or not
if (NOT DEFINED BUILD_ARM64)
    set (BUILD_ARM64 true)
endif ()
set (BUILD_ARM64 ${BUILD_ARM64} CACHE STRING "Build arm64 arch or not")

# Check the platform selection and setup for developer root
if (${IOS_PLATFORM} STREQUAL "OS")
	set (IOS_PLATFORM_LOCATION "iPhoneOS.platform")

	# This causes the installers to properly locate the output libraries
	set (CMAKE_XCODE_EFFECTIVE_PLATFORMS "-iphoneos")
elseif (${IOS_PLATFORM} STREQUAL "SIMULATOR")
    set (IS_SIMULATOR true)
	set (IOS_PLATFORM_LOCATION "iPhoneSimulator.platform")

	# This causes the installers to properly locate the output libraries
	set (CMAKE_XCODE_EFFECTIVE_PLATFORMS "-iphonesimulator")
elseif (${IOS_PLATFORM} STREQUAL "CATALYST")
	set (IOS_PLATFORM_LOCATION "MacOSX.platform")
else ()
	message (FATAL_ERROR "Unsupported IOS_PLATFORM value selected. Please choose OS or SIMULATOR")
endif ()

# Setup iOS developer location unless specified manually with CMAKE_IOS_DEVELOPER_ROOT
# Note Xcode 4.3 changed the installation location, choose the most recent one available
exec_program(/usr/bin/xcode-select ARGS -print-path OUTPUT_VARIABLE CMAKE_XCODE_DEVELOPER_DIR)
set (XCODE_POST_43_ROOT "${CMAKE_XCODE_DEVELOPER_DIR}/Platforms/${IOS_PLATFORM_LOCATION}/Developer")
set (XCODE_PRE_43_ROOT "/Developer/Platforms/${IOS_PLATFORM_LOCATION}/Developer")
if (NOT DEFINED CMAKE_IOS_DEVELOPER_ROOT)
	if (EXISTS ${XCODE_POST_43_ROOT})
		set (CMAKE_IOS_DEVELOPER_ROOT ${XCODE_POST_43_ROOT})
	elseif(EXISTS ${XCODE_PRE_43_ROOT})
		set (CMAKE_IOS_DEVELOPER_ROOT ${XCODE_PRE_43_ROOT})
	endif (EXISTS ${XCODE_POST_43_ROOT})
endif ()
set (CMAKE_IOS_DEVELOPER_ROOT ${CMAKE_IOS_DEVELOPER_ROOT} CACHE PATH "Location of iOS Platform")

# Find and use the most recent iOS sdk unless specified manually with CMAKE_IOS_SDK_ROOT
if (NOT DEFINED CMAKE_IOS_SDK_ROOT)
	file (GLOB _CMAKE_IOS_SDKS "${CMAKE_IOS_DEVELOPER_ROOT}/SDKs/*")
	if (_CMAKE_IOS_SDKS) 
		list (SORT _CMAKE_IOS_SDKS)
		list (REVERSE _CMAKE_IOS_SDKS)
		list (GET _CMAKE_IOS_SDKS 0 CMAKE_IOS_SDK_ROOT)
	else (_CMAKE_IOS_SDKS)
		message (FATAL_ERROR "No iOS SDK's found in default search path ${CMAKE_IOS_DEVELOPER_ROOT}. Manually set CMAKE_IOS_SDK_ROOT or install the iOS SDK.")
	endif (_CMAKE_IOS_SDKS)
	message (STATUS "Toolchain using default iOS SDK: ${CMAKE_IOS_SDK_ROOT}")
endif ()
set (CMAKE_IOS_SDK_ROOT ${CMAKE_IOS_SDK_ROOT} CACHE PATH "Location of the selected iOS SDK")

# Set the sysroot default to the most recent SDK
set (CMAKE_OSX_SYSROOT ${CMAKE_IOS_SDK_ROOT} CACHE PATH "Sysroot used for iOS support")

# set the architecture for iOS 
if (${IOS_PLATFORM} STREQUAL "OS")
    set (IOS_ARCH armv7;arm64)
elseif (${IOS_PLATFORM} STREQUAL "SIMULATOR" OR ${IOS_PLATFORM} STREQUAL "CATALYST")
    set (IOS_ARCH x86_64)
endif ()

set (CMAKE_OSX_ARCHITECTURES ${IOS_ARCH} CACHE STRING  "Build architecture for iOS")

# Set the find root to the iOS developer roots and to user defined paths
set (CMAKE_FIND_ROOT_PATH ${CMAKE_IOS_DEVELOPER_ROOT} ${CMAKE_IOS_SDK_ROOT} ${CMAKE_PREFIX_PATH} CACHE STRING  "iOS find search path root")

# default to searching for frameworks first
set (CMAKE_FIND_FRAMEWORK FIRST)

# set up the default search directories for frameworks
set (CMAKE_SYSTEM_FRAMEWORK_PATH
	${CMAKE_IOS_SDK_ROOT}/System/Library/Frameworks
	${CMAKE_IOS_SDK_ROOT}/System/Library/PrivateFrameworks
	${CMAKE_IOS_SDK_ROOT}/Developer/Library/Frameworks
)

# only search the iOS sdks, not the remainder of the host filesystem
set (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)
set (CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set (CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)


# This little macro lets you set any XCode specific property
macro (set_xcode_property TARGET XCODE_PROPERTY XCODE_VALUE)
	set_property (TARGET ${TARGET} PROPERTY XCODE_ATTRIBUTE_${XCODE_PROPERTY} ${XCODE_VALUE})
endmacro (set_xcode_property)


# This macro lets you find executable programs on the host system
macro (find_host_package)
	set (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
	set (CMAKE_FIND_ROOT_PATH_MODE_LIBRARY NEVER)
	set (CMAKE_FIND_ROOT_PATH_MODE_INCLUDE NEVER)
	set (IOS FALSE)

	find_package(${ARGN})

	set (IOS TRUE)
	set (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)
	set (CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
	set (CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
endmacro (find_host_package)
{{< /highlight >}}

With support for Catalyst now enabled for CMake, I created a Bash shell script to automate creating the XCFramework for libssh2:

{{< highlight bash "linenos=table" >}}
SCRIPT_DIR=$(dirname $0)
pushd $SCRIPT_DIR/.. > /dev/null
ROOT_PATH=$PWD
popd > /dev/null

PLATFORMS="OS SIMULATOR CATALYST"
for PLATFORM in $PLATFORMS
do
    echo "Building libssh2 for $PLATFORM"

    rm -rf /tmp/libssh2
    cp -r External/libssh2 /tmp/

    pushd /tmp/libssh2 > /dev/null

    LOG=/tmp/libssh2-$PLATFORM.log
    rm -f $LOG

    OUTPUT_PATH=$ROOT_PATH/build/libssh2/$PLATFORM
    rm -rf $OUTPUT_PATH

    case $PLATFORM in
        "OS" )
            OPENSSL_ROOT_DIR=$ROOT_PATH/build/openssl/ios
            OPENSSL_CRYPTO_LIBRARY=$ROOT_PATH/build/openssl/lib/libcrypto.a
            OPENSSL_SSL_LIBRARY=$ROOT_PATH/build/openssl/lib/libssl.a
            ;;

        "SIMULATOR" )
            OPENSSL_ROOT_DIR=$ROOT_PATH/build/openssl/iossimulator
            OPENSSL_CRYPTO_LIBRARY=$OPENSSL_ROOT_DIR/lib/libcrypto.a
            OPENSSL_SSL_LIBRARY=$OPENSSL_ROOT_DIR/lib/libssl.a
            ;;

        "CATALYST" )
            OPENSSL_ROOT_DIR=$ROOT_PATH/build/openssl/catalyst
            OPENSSL_CRYPTO_LIBRARY=$OPENSSL_ROOT_DIR/lib/libcrypto.a
            OPENSSL_SSL_LIBRARY=$OPENSSL_ROOT_DIR/lib/libssl.a
            ;;
    esac

    OPENSSL_INCLUDE_DIR=$OPENSSL_ROOT_DIR/include

    mkdir bin
    cd bin
    cmake \
        -DCMAKE_TOOLCHAIN_FILE=$ROOT_PATH/External/cmake/iOS.cmake \
        -DIOS_PLATFORM=$PLATFORM \
        -DCMAKE_INSTALL_PREFIX=$OUTPUT_PATH \
        -DCRYPTO_BACKEND=OpenSSL \
        -DOPENSSL_ROOT_DIR=$OPENSSL_ROOT_DIR \
        -DOPENSSL_CRYPTO_LIBRARY=$OPENSSL_CRYPTO_LIBRARY \
        -DOPENSSL_SSL_LIBRARY=$OPENSSL_SSL_LIBRARY \
        -DOPENSSL_INCLUDE_DIR=$OPENSSL_INCLUDE_DIR \
        .. >> $LOG 2>&1
    cmake --build . --target install >> $LOG 2>&1

    popd > /dev/null
done

echo "Creating the XCFramework"

LIB_PATH=$ROOT_PATH/lib/libssh2
LIBSSH2_PATH=$LIB_PATH/libssh2.xcframework
rm -rf $LIBSSH2_PATH
mkdir -p $LIB_PATH

xcodebuild -create-xcframework \
    -library $ROOT_PATH/build/libssh2/OS/lib/libssh2.a \
    -headers $ROOT_PATH/build/libssh2/OS/include \
    -library $ROOT_PATH/build/libssh2/SIMULATOR/lib/libssh2.a \
    -headers $ROOT_PATH/build/libssh2/SIMULATOR/include \
    -library $ROOT_PATH/build/libssh2/CATALYST/lib/libssh2.a \
    -headers $ROOT_PATH/build/libssh2/CATALYST/include \
    -output $LIBSSH2_PATH

pushd $LIB_PATH > /dev/null
zip -r ../libssh2.zip .
popd > /dev/null

echo "Done; cleaning up"
rm -rf /tmp/libssh2
{{< /highlight >}}

I can run this script to generate the XCFramework for libssh2:

    bin/build_libssh2.sh

The end result is that the `lib/libssh2.zip` archive now exists.

## libgit2

With OpenSSL and libssh2 built, building libgit2 is fairly easy to build. Like libssh2, libgit2 also uses CMake, so the earlier configuration that I created with support for Catalyst can be reused. I created a Bash script to automate building libgit2 and packaging it as an XCFramework:

{{< highlight bash "linenos=table" >}}
SCRIPT_DIR=$(dirname $0)
pushd $SCRIPT_DIR/.. > /dev/null
ROOT_PATH=$PWD
popd > /dev/null

PLATFORMS="OS SIMULATOR CATALYST"
for PLATFORM in $PLATFORMS
do
    echo "Building libgit2 for $PLATFORM"

    rm -rf /tmp/libgit2
    cp -r External/libgit2 /tmp/

    pushd /tmp/libgit2 > /dev/null

    LOG=/tmp/libgit2-$PLATFORM.log
    rm -f $LOG

    OUTPUT_PATH=$ROOT_PATH/build/libgit2/$PLATFORM
    rm -rf $OUTPUT_PATH

    case $PLATFORM in
        "OS" )
            OPENSSL_ROOT_DIR=$ROOT_PATH/build/openssl/ios
            OPENSSL_LIBRARIES_DIR=$ROOT_PATH/build/openssl/lib
            ;;

        "SIMULATOR" )
            OPENSSL_ROOT_DIR=$ROOT_PATH/build/openssl/iossimulator
            OPENSSL_LIBRARIES_DIR=$OPENSSL_ROOL_DIR/lib
            ;;

        "CATALYST" )
            OPENSSL_ROOT_DIR=$ROOT_PATH/build/openssl/catalyst
            OPENSSL_LIBRARIES_DIR=$OPENSSL_ROOT_DIR/lib
            ;;
    esac

    OPENSSL_INCLUDE_DIR=$OPENSSL_ROOT_DIR/include
    OPENSSL_CRYPTO_LIBRARY=$OPENSSL_LIBRARIES_DIR/libcrypto.a
    OPENSSL_SSL_LIBRARY=$OPENSSL_LIBRARIES_DIR/libssl.a
    LIBSSH2_ROOT_DIR=$ROOT_PATH/build/libssh2/$PLATFORM

    mkdir bin
    cd bin
    cmake \
        -DCMAKE_TOOLCHAIN_FILE=$ROOT_PATH/External/cmake/iOS.cmake \
        -DIOS_PLATFORM=$PLATFORM \
        -DCMAKE_INSTALL_PREFIX=$OUTPUT_PATH \
        -DOPENSSL_ROOT_DIR=$OPENSSL_ROOT_DIR \
        -DOPENSSL_CRYPTO_LIBRARY=$OPENSSL_CRYPTO_LIBRARY \
        -DOPENSSL_SSL_LIBRARY=$OPENSSL_SSL_LIBRARY \
        -DOPENSSL_INCLUDE_DIR=$OPENSSL_INCLUDE_DIR \
        -DUSE_SSH=OFF \
        -DLIBSSH2_FOUND=TRUE \
        -DLIBSSH2_INCLUDE_DIRS=$LIBSSH2_ROOT_DIR/include \
        -DLIBSSH2_LIBRARY_DIRS=$LIBSSH2_ROOT_DIR/lib \
        -DLIBSSH2_LIBRARIES="-L$LIBSSH2_ROOT_DIR/lib -L$OPENSSL_LIBRARIES_DIR -lssh2 -lssl -lcrypto" \
        -DBUILD_SHARED_LIBS=OFF \
        -DBUILD_CLAR=OFF \
        .. >> $LOG 2>&1
    cmake --build . --target install >> $LOG 2>&1

    popd > /dev/null
done

echo "Creating the XCFramework"

LIB_PATH=$ROOT_PATH/lib/libgit2
LIBGIT2_PATH=$LIB_PATH/libgit2.xcframework
rm -rf $LIBGIT2_PATH
mkdir -p $LIB_PATH

xcodebuild -create-xcframework \
    -library $ROOT_PATH/build/libgit2/OS/lib/libgit2.a \
    -headers $ROOT_PATH/build/libgit2/OS/include \
    -library $ROOT_PATH/build/libgit2/SIMULATOR/lib/libgit2.a \
    -headers $ROOT_PATH/build/libgit2/SIMULATOR/include \
    -library $ROOT_PATH/build/libgit2/CATALYST/lib/libgit2.a \
    -headers $ROOT_PATH/build/libgit2/CATALYST/include \
    -output $LIBGIT2_PATH

pushd $LIB_PATH > /dev/null
zip -r ../libgit2.zip .
popd > /dev/null

echo "Done; cleaning up"
rm -rf /tmp/libgit2
{{< /highlight >}}

I can run this script to create the XCFramework:

    bin/build_libgit2.sh

The end result is that the `lib/libgit2.zip` archive will be produced.

## Redistributing libgit2

Now that I have created binary XCFrameworks for libgit2, libssh2, and OpenSSL, I can distribute these files as a Swift Package and they can be consumed using Swift Package Manager. Since the frameworks are redistributed in binary form, I just have to include the download URLs for the binary frameworks in the `Package.swift` file. Because the main target of this article is libgit2, I am going to use the libgit2 version number as the version number for my release that I will host on GitHub.

{{< highlight swift "linenos=table" >}}
// swift-tools-version:5.3

import PackageDescription

let package = Package(
  name: "libgit2-ios",
  platforms: [.iOS(.v13)],
  products: [
    .library(
      name: "libgit2",
      targets: [
        "libgit2",
        "libssh2",
        "libssl",
        "libcrypto"
      ]
    )
  ],
  targets: [
    .binaryTarget(
      name: "libgit2",
      url: "https://github.com/mfcollins3/libgit2-ios/releases/download/v1.1.0/libgit2.zip",
      checksum: "41d9e4efe604059abb5e1eaa4f8af52a1f19d22b5e103037eed0af3b95ce96ab"
    ),
    .binaryTarget(
      name: "libssh2",
      url: "https://github.com/mfcollins3/libgit2-ios/releases/download/v1.1.0/libssh2.zip",
      checksum: "bc1a6c68e7e50ca9370e516add2f1baa814c0a7d00ef19ee5278862f1ceb2923"
    ),
    .binaryTarget(
      name: "libssl",
      url: "https://github.com/mfcollins3/libgit2-ios/releases/download/v1.1.0/libssl.zip",
      checksum: "6f9e8f58404e5a3e5cb9ab913f3a6320b6730c0f578159a1b366f9c4ba72c31c"
    ),
    .binaryTarget(
      name: "libcrypto",
      url: "https://github.com/mfcollins3/libgit2-ios/releases/download/v1.1.0/libcrypto.zip",
      checksum: "09ec647ae24e959c43a6c1321717ae9a3d56b53aa99c51a7826311b86c12d913"
    )
  ]
)
{{< /highlight >}}

The checksums of the ZIP archives for the frameworks can be generated using the `swift package compute-checksum` command:

    swift package compute-checksum lib/libgit2.zip
    swift package compute-checksum lib/libssh2.zip
    swift package compute-checksum lib/libssl.zip
    swift package compute-checksum lib/libcrypto.zip

With `Package.swift` created, I created a release on GitHub and tagged the source code with the tag `v1.1.0` to match the libgit2 version. I then uploaded `libgit2.zip`, `libssh2.zip`, `libssl.zip`, and `libcrypto.zip` to GitHub and added them to the release.

## Consuming libgit2

With libgit2 now packaged as a Swift package and hosted publicly on GitHub, I can now add libgit2 to my Naked Blogging application. I started by opening my workspace in Xcode and navigated to the Project settings for my Blogging application project. Under **Frameworks, Libraries, and Embedded Content**, I added a framework and chose to add a package dependency. I used the URL `https://github.com/mfcollins3/libgit2-ios.git` as the package repository URL. Xcode scanned the repository and chose the 1.1.0 release automatically for me. After accepting that option, the package was added to my application project and Xcode and Swift Package Manager downloaded the binary frameworks and made them available to my application.

{{< admonition type="note" title="Additional Dependencies" open="true" >}}
In order to successfully build the application, you will need to include additional system libraries in your application that are required by libgit2. Please add the following system libraries before building:

* libiconv
* libz
{{< /admonition >}}

Because libgit2 is a C library, in order to consume the framework I need to create a [bridging header](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/importing_objective-c_into_swift) in my Swift application. This can be done easily by adding a dummy Objective-C source file and then deleting the source file. The bridging header will automatically be generated and will remain in the project. All that I need to do in the bridging header is to import the `git2.h` header file, and the libgit2 API will be available to my application:

{{< highlight c "linenos=table" >}}
#import <git2.h>
{{< /highlight >}}

Before I can make any calls to libgit2, I need to initialize libgit2 by calling the [`git_libgit2_init`](https://libgit2.org/libgit2/#HEAD/group/libgit2/git_libgit2_init) API. I want to do this when the application launches, so I am going to create a custom application delegate will call `git_libgit2_init` in the `application(_:didFinishLaunchingWithOptions:)` method:

{{< highlight swift "linenos=table" >}}
import UIKit

typealias LaunchOptions = [UIApplication.LaunchOptionsKey: Any]

final class AppDelegate: UIResponder, UIApplicationDelegate {
  func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: LaunchOptions? = nil
  ) -> Bool {
    git_libgit2_init()

    return true
  }
}
{{< /highlight >}}

I then registered the custom application delegate in my SwiftUI `App` type:

{{< highlight swift "linenos=table" >}}
import SwiftUI

@main
struct BloggingApp: App {
  @UIApplicationDelegateAdaptor private var appDelegate: AppDelegate

  var body: some Scene {
    WindowGroup {
      ContentView()
    }
  }
}
{{< /highlight >}}

If I build and run and set a breakpoint in my application delegate, I should see that the call to `git_libgit2_init` succeeds and my application launches.

At this point, I can use any of the libgit2 APIs, however, as I need to use the APIs, I will probably create Swift wrappers around them. The libgit2 APIs are after all C APIs, and they are not necessarily the most friendliest (or "Swift"-iest) to use, so I will wrap them in Swift types to make them better for Swift programming.

I'm going to test out my integrations by adding the ability to create a local repository and then open the local repository. I will start by creating a `GitRepository` class that wraps the libgit2 repository API. Using Swift's C interoperability, libgit2 uses [`OpaquePointer`](https://developer.apple.com/documentation/swift/opaquepointer#) values to represent Git objects which are pointers in C. My `GitRepository` type will wrap the repository's `OpaquePointer` and will use it to call the libgit2 APIs to operate on or query the repository. libgit2's repository API also relies on file paths, while a typical iOS/Swift application will use file URLs to reference files in the file system. So in my custom `GitRepository` class, I will handle that transition so that consumers use `URL` types and I translate them to strings or back as necessary.

{{< highlight swift "linenos=table" >}}
import Foundation

final class GitRepository {
  private let repository: OpaquePointer

  var url: URL {
    let path = String(cString: git_repository_path(repository))
    return URL(fileURLWithPath: path, isDirectory: true)
  }

  var workURL: URL? {
    guard let workDir = git_repository_workdir(repository) else {
      return nil
    }

    let path = String(cString: workDir)
    return URL(fileURLWithPath: path, isDirectory: true)
  }

  init(at url: URL, isBare: Bool = false) throws {
    var pointer: OpaquePointer?
    let result = git_repository_init(&pointer, url.path, isBare ? 1 : 0)
    guard result == 0 else {
      fatalError("Unexpected libgit2 error: \(result))
    }

    self.repository = pointer!
  }

  init(open url: URL) throws {
    var pointer: OpaquePointer?
    let result = git_repository_open(&pointer, url.path)
    guard result == 0 else {
      fatalError("Unexpected libgit2 error: \(result)")
    }

    self.repository = pointer!
  }

  deinit {
    git_repository_free(repository)
  }
}
{{< /highlight >}}

A couple of things to note about this implementation. A Git repository can either be bare or non-bare. Typically, a bare repository is a main repository that is found on a Git server like GitHub. What most developers see if a work directory with a hidden `.git` directory containing the local copy of the repository. You will notice that the `workURL` property returns an optional URL. If the Git repository is a bare repository, then the call to `git_repository_workdir` will return a C `NULL` which will be translated into a Swift `nil` value. If the repository is bare, `workURL` will return a `nil` value, otherwise `workURL` will return the file URL for the work directory for the repository.

The second thing to notice is that my initializers are annotated with `throws` but do not throw anything. In a later post, I will dive into the APIs more and implement error handling and throwing exceptions. I put those in for now as a placeholder.

The final thing to notice in the code is the `deinit` deinitializer. When the `GitRepository` object is no longer being referenced, `deinit` will run and it will use the `git_repository_free` API to release the memory being used by the underlying Git repository. Remember that this is C code that we are interfacing with and it does not support [ARC](https://en.wikipedia.org/wiki/Automatic_Reference_Counting), so we need to make sure that we are cleaning up and freeing memory and Git objects when no longer necessary.

I updated my `ContentView` view type temporarily to test out creating or opening a test Git repository that is stored in my application's documents directory:

{{< highlight swift "linenos=table" >}}
import Foundation
import SwiftUI

struct ContentView: View {
  var body: some View {
    VStack {
      Button("Create Repository") {
        let fileManager = FileManager.default
        do {
          let documentsURL = try fileManager.url(
            for: .documentDirectory,
            in: .userDomainMask,
            appropriateFor: nil,
            create: true
          )
          let repositoryURL = documentsURL.appendingPathComponent(
            "test",
            isDirectory: true
          )
          if fileManager.fileExists(atPath: repositoryURL.path, isDirectory: nil) {
            try fileManager.removeItem(at: repositoryURL)
          }

          let repository = try GitRepository(at: repositoryURL)
          print("Repository created at \(repository.url)")
          if let workURL = repository.workURL {
            print("Work directory at \(workURL)")
          }
        } catch {
          print("ERROR: \(error)")
        }
      }
      .padding()

      Button("Open Repository") {
        do {
          let documentsURL = try FileManager.default.url(
            for: .documentDirectory,
            in: .userDomainMask,
            appropriateFor: nil,
            create: true
          )
          let repositoryURL = documentsURL.appendingPathComponent(
            "test",
            isDirectory: true
          )

          let repository = try GitRepository(open: repositoryURL)
          print("Repository opened from \(repository.url)")
          if let workURL = repository.workURL {
            print("Work directory at \(workURL)"))
          }
        } catch {
          print("ERROR: \(error)")
        }
      }
      .padding()
    }
  }
}
{{< /highlight >}}

When I run this in the simulator or on a device, I get my two buttons. I can see in the debugger that tapping the **Create Repository** button will successfully create the test repository and that tapping the **Open Repository** button will successfully open the test repository. Putting a breakpoint on the `GitRepository` deinitializer shows me that when the `GitRepository` object goes out of scope, the deinitializer is called and the underlying Git repository object is freed.

## Summing It Up

In this article, I showed you how I built libgit2, libssh2, and OpenSSL from source code so that I can use them in my applications. I packaged libgit2, libssh2, and OpenSSL in XCFrameworks so that I can distribute them in binary form and make them easily reusable in my applications. I then packaged the XCFrameworks in a Swift Package so that I can consume the XCFrameworks using Swift Package Manager. I then demonstrated how to consume libgit2 from a Swift application.

In future posts, I will develop out my Swift wrapper API over libgit2 as I integrate Git functionality into my application. There's a lot of content and use cases to cover and I don't want to write another really long post (this post already seems long enough).

If you want to try my libgit2 frameworks with your own application, it is hosted in on GitHub at [mfcollins3/libgit2-ios](https://github.com/mfcollins3/libgit2-ios).