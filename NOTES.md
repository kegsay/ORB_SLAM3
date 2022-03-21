## ORB-SLAM3 notes

Next time:
- bus error is preventing things from working.

terminal A:
- ORB_SLAM3/build
- cmake .. -DCMAKE_BUILD_TYPE=Release
- make -j8

terminal B:
- ORB_SLAM3/Examples
- ./Monocular/mono_euroc ../Vocabulary/ORBShortvoc.txt ./Monocular/EuRoC.yaml /Users/kegan/wall-e/orbslam3/mh01 ./Monocular/EuRoC_TimeStamps/MH01.txt dataset-MH01_mono

change shortvoc to voc when it works.

Cause appears to be in `Tracking.cc` as `CreateInitialMapMonocular` cannot be called (console log works before, but doesn't work inside)

Possible alignment issues? https://github.com/PointCloudLibrary/pcl/issues/4258
running in debug mode no worky (removes `-march=native`)

:SSSSS


ASan shows stack overflow:

AddressSanitizer:DEADLYSIGNAL
=================================================================
==45462==ERROR: AddressSanitizer: stack-overflow on address 0x700006495f18 (pc 0x000104ad0618 bp 0x7000065282f0 sp 0x700006495ec0 T1)
    #0 0x104ad0617 in ORB_SLAM3::Tracking::GrabImageMonocular(cv::Mat const&, double const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >) Tracking.cc:1567
    #1 0x1049f213e in ORB_SLAM3::System::TrackMonocular(cv::Mat const&, double const&, std::__1::vector<ORB_SLAM3::IMU::Point, std::__1::allocator<ORB_SLAM3::IMU::Point> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >) System.cc:477
    #2 0x103d8dcd7 in void* std::__1::__thread_proxy<std::__1::tuple<std::__1::unique_ptr<std::__1::__thread_struct, std::__1::default_delete<std::__1::__thread_struct> >, main::$_0> >(void*) thread:351
    #3 0x7ff8166064e0 in _pthread_start (libsystem_pthread.dylib:x86_64+0x64e0)
    #4 0x7ff816601f6a in thread_start (libsystem_pthread.dylib:x86_64+0x1f6a)

SUMMARY: AddressSanitizer: stack-overflow Tracking.cc:1567 in ORB_SLAM3::Tracking::GrabImageMonocular(cv::Mat const&, double const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >)
Thread T1 created by T0 here:
    #0 0x10a560b8a in wrap_pthread_create (libclang_rt.asan_osx_dynamic.dylib:x86_64h+0x3fb8a)
    #1 0x103d801a6 in main mono_euroc.cc:90
    #2 0x10641e51d in start (dyld:x86_64+0x551d)

==45462==ABORTING

### Installation

- Commit `4452a3c4ab75b1cde34e5505a36ec3f9edcdc4c4` on https://github.com/UZ-SLAMLab/ORB_SLAM3 (also forked)
- Install Pangolin from source, see github.
- Grab an eigen tarball and unzip it, see github. I built it with cmake so it seemed to pick up the lib?
- `brew install opencv` :D
- Run build.sh once (it isn't idempotent), it'll fail.

Set clang as your compiler:
```
export CXX=/usr/bin/clang++
export CC=/usr/bin/clang
```

#### DBoW

- Remove '-gcc' in import: `ORB_SLAM3/Thirdparty/DBoW2/DBoW2/FORB.cpp:16:10: fatal error: 'stdint-gcc.h' file not found`
- Ensure boost is picked up, modify `ORB_SLAM3/Thirdparty/DBoW2/CMakeLists.txt` (anywhere?):
     ```
     find_package(Boost)
     include_directories( ${Boost_INCLUDE_DIRS} )
     ```

Then run:
```
cd Thirdparty/DBoW2
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j
```

#### g2o

g2o needs to have `<tr/xxxx>` imports stripped of `tr1`:
 - `find g2o -iname "*.h" | xargs sed -i '' 's$tr1/$$g'`
 - `find g2o -iname "*.h" | xargs sed -i '' 's$std::tr1$std$g'`

Then run:
```
cd Thirdparty/g2o
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j
```

#### Sophus

The only one which works without modification:

```
cd Thirdparty/Sophus
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j
```

#### ORB-SLAM3

Finally we can build ORB-SLAM3.

tl;dr Modify `CMakeLists.txt`:
```
+set(OPENSSL_ROOT_DIR "/usr/local/opt/openssl")
+find_package(openssl REQUIRED)
+
+find_package(Boost COMPONENTS serialization REQUIRED)

include_directories(
...
 ${Pangolin_INCLUDE_DIRS}
+${OPENSSL_INCLUDE_DIR}


-${PROJECT_SOURCE_DIR}/Thirdparty/DBoW2/lib/libDBoW2.so
-${PROJECT_SOURCE_DIR}/Thirdparty/g2o/lib/libg2o.so
--lboost_serialization
--lcrypto
+${PROJECT_SOURCE_DIR}/Thirdparty/DBoW2/lib/libDBoW2.dylib
+${PROJECT_SOURCE_DIR}/Thirdparty/g2o/lib/libg2o.dylib
+${OPENSSL_CRYPTO_LIBRARY}
+${Boost_SERIALIZATION_LIBRARY}
```

Add crap to make OpenMP be used:
```
set(OpenMP_CXX "${CMAKE_CXX_COMPILER}")
set(OpenMP_CXX_FLAGS "-fopenmp=libomp -Wno-unused-command-line-argument")
set(OpenMP_CXX_LIB_NAMES "libomp" "libgomp" "libiomp5")
set(OpenMP_libomp_LIBRARY ${OpenMP_CXX_LIB_NAMES})
set(OpenMP_libgomp_LIBRARY ${OpenMP_CXX_LIB_NAMES})
set(OpenMP_libiomp5_LIBRARY ${OpenMP_CXX_LIB_NAMES})
set(OpenMP_C "${CMAKE_C_COMPILER}")
set(OpenMP_C_FLAGS "-fopenmp=libomp -Wno-unused-command-line-argument")
set(OpenMP_C_LIB_NAMES "libomp" "libgomp" "libiomp5")
set(OpenMP_libomp_LIBRARY ${OpenMP_C_LIB_NAMES})
set(OpenMP_libgomp_LIBRARY ${OpenMP_C_LIB_NAMES})
set(OpenMP_libiomp5_LIBRARY ${OpenMP_C_LIB_NAMES})
```

- Strip `-gcc`: `/ORB_SLAM3/src/ORBmatcher.cc:28:9: fatal error: 'stdint-gcc.h' file not found`
- It needs openssl for MD5: `brew install openssl`
- Need to tell CMake to find the files in `ORB_SLAM3/CMakeLists.txt`:
    ```
    find_package(openssl REQUIRED)
    include_directories(${OPENSSL_INCLUDE_DIR})
    ```
- But you'll get `Could NOT find OpenSSL, try to set the path to OpenSSL root folder in the system variable OPENSSL_ROOT_DIR (missing: OPENSSL_CRYPTO_LIBRARY OPENSSL_INCLUDE_DIR) `
- So tell CMake some more to find the files (just above the find_package):
    ```
    set(OPENSSL_ROOT_DIR "/usr/local/opt/openssl")
    ```
- Dynamic linking will fail as it looks for `.so` but we are making `.dylib`, so rename `.so` to `.dylib`:
    ```
    ${PROJECT_SOURCE_DIR}/Thirdparty/DBoW2/lib/libDBoW2.dylib
    ${PROJECT_SOURCE_DIR}/Thirdparty/g2o/lib/libg2o.dylib
    ```
- `target_link_libraries` has things like `-lcrypto` and `-lboost_serialization` which are evil: https://cmake.org/pipermail/cmake/2018-April/067417.html
  so instead, as we have `find_package`d them, use them instead:
    ```
    -lcrypto
    -lboost_serialization
    ```
    into:
    ```
    ${OPENSSL_CRYPTO_LIBRARY}
    ${Boost_SERIALIZATION_LIBRARY}
    ```

Now compile it:
```
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4
```