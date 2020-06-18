# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.4.1)

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds them for you.
# Gradle automatically packages shared libraries with your APK.

add_library( # Sets the name of the library.
        native-lib

        # Sets the library as a shared library.
        SHARED

        # Provides a relative path to your source file(s).
        native-lib.cpp)

# Searches for a specified prebuilt library and stores the path as a
# variable. Because CMake includes system libraries in the search path by
# default, you only need to specify the name of the public NDK library
# you want to add. CMake verifies that the library exists before
# completing its build.

find_library( # Sets the name of the path variable.
        log-lib

        # Specifies the name of the NDK library that
        # you want CMake to locate.
        log)

# Specifies libraries CMake should link to your target library. You
# can link multiple libraries, such as libraries you define in this
# build script, prebuilt third-party libraries, or system libraries.

target_link_libraries( # Specifies the target library.
        native-lib

        # Links the target library to the log library
        # included in the NDK.
        ${log-lib})

add_executable(ip-up-vpn
        ip-up-vpn.c)
target_link_libraries(ip-up-vpn
        ${log-lib})

message("ANDROID_SDK_ROOT: ${ANDROID_SDK_ROOT}") # ANDROID_SDK_ROOT should be passed as variable to this script.
find_program(ADB NAMES adb PATHS ${ANDROID_SDK_ROOT}/platform-tools)

add_library(foo
        SHARED
        foo.cpp foo_jni.cpp)
target_link_libraries(foo
        ${log-lib})

# add gtest
set(GOOGLETEST_ROOT ${ANDROID_NDK}/sources/third_party/googletest)
add_library(gtest STATIC ${GOOGLETEST_ROOT}/src/gtest_main.cc ${GOOGLETEST_ROOT}/src/gtest-all.cc)
target_include_directories(gtest PRIVATE ${GOOGLETEST_ROOT})
target_include_directories(gtest PUBLIC ${GOOGLETEST_ROOT}/include)

add_executable(footest test/foo_unittest.cpp foo.cpp)
target_link_libraries(footest gtest)

#find_program(ADB adb)
add_custom_command(TARGET footest POST_BUILD COMMENT "POST_BUILD" VERBATIM
        COMMAND ${ADB} shell mkdir -p /data/local/tmp/${ANDROID_ABI}
        COMMAND ${ADB} push $<TARGET_FILE:native-lib> /data/local/tmp/${ANDROID_ABI}/
        COMMAND ${ADB} push $<TARGET_FILE:ip-up-vpn> /data/local/tmp/${ANDROID_ABI}/
        COMMAND ${ADB} push $<TARGET_FILE:footest> /data/local/tmp/${ANDROID_ABI}/
        COMMAND ${ADB} shell "chmod a+x /data/local/tmp/${ANDROID_ABI}/footest"
        COMMAND ${ADB} shell "LD_LIBRARY_PATH=/data/local/tmp/${ANDROID_ABI} /data/local/tmp/${ANDROID_ABI}/footest")
#        COMMAND ${ADB} shell "LD_LIBRARY_PATH=/data/local/tmp/${ANDROID_ABI} chmod a+x /data/local/tmp/${ANDROID_ABI}/footest /data/local/tmp/${ANDROID_ABI}/footest")

# go_jni test

## this is for jni indexing
add_library(go_jni go_jni/go_jni.c)

string(TOLOWER ${CMAKE_HOST_SYSTEM_NAME} OS)
string(TOLOWER ${CMAKE_HOST_SYSTEM_PROCESSOR} ARCH)

## custom go build for go_jni
add_custom_target(libgo_jni_${VERSION}.so WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/go_jni" COMMENT "Building go_jni" VERBATIM COMMAND
        ${ANDROID_NDK}/prebuilt/${OS}-${ARCH}/bin/make
        ANDROID_ARCH_NAME=${ANDROID_ARCH_NAME}
        ANDROID_C_COMPILER=${ANDROID_C_COMPILER}
        ANDROID_TOOLCHAIN_ROOT=${ANDROID_TOOLCHAIN_ROOT}
        ANDROID_LLVM_TRIPLE=${ANDROID_LLVM_TRIPLE}
        ANDROID_SYSROOT=${ANDROID_SYSROOT}
        ANDROID_PACKAGE_NAME=${ANDROID_PACKAGE_NAME}
        VERSION=${VERSION}
        GRADLE_USER_HOME=${GRADLE_USER_HOME}
        CFLAGS=${CMAKE_C_FLAGS}\ -Wno-unused-command-line-argument
        LDFLAGS=${CMAKE_SHARED_LINKER_FLAGS}\ -fuse-ld=gold
        DESTDIR=${CMAKE_LIBRARY_OUTPUT_DIRECTORY}
        BUILDDIR=${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/../generated-src
        )

add_dependencies(footest libgo_jni_${VERSION}.so)