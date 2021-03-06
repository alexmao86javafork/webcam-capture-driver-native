cmake_minimum_required(VERSION 3.5.1)

set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LANGUAGE_STANDARD "c++11")
set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LIBRARY "libc++")

set(CMAKE_CONFIGURATION_TYPES "Release" CACHE STRING "Build as Release by default")
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.8" CACHE STRING "Support OS X down to 10.8 (earlier versions not tested)")

project (WebcamCaptureDriverNative)

if (CMAKE_SIZEOF_VOID_P EQUAL 8)
	set(bitness 64)
elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
	set(bitness 32)
else()
	message(FATAL_ERROR "Unable to identify bitness of target architecture")
endif()

message(STATUS "QUERY JAVA_HOME: ${JAVA_HOME}")
message(STATUS "QUERY ENV_JAVA_HOME: $ENV{JAVA_HOME}")
find_package(JNI REQUIRED)
find_package(Java 1.7 COMPONENTS Development REQUIRED)
include(UseJava)
message(STATUS "Found Java_JAVA_EXECUTABLE: ${Java_JAVA_EXECUTABLE}")
message(STATUS "Found JNI_INCLUDE_DIRS: ${JNI_INCLUDE_DIRS}")
SET(CMAKE_JAVA_COMPILE_FLAGS ${CMAKE_JAVA_COMPILE_FLAGS} -source 1.7 -target 1.7)


set(libyuv-path  "libyuv")
set(video-capture-library-path  "video-capture-library")
set(webcam-capture-jar "${CMAKE_CURRENT_SOURCE_DIR}/java/extern/webcam-capture-0.3.10.jar")
set(slf4j-jar "${CMAKE_CURRENT_SOURCE_DIR}/java/extern/slf4j-api-1.7.2.jar")

enable_testing()

set(USE_GENERATE_X86 ON CACHE BOOL "Currently OS X and Windows supported: Therefore ON by default")
add_subdirectory(${video-capture-library-path}/build)

add_subdirectory(${libyuv-path})


add_jar(webcam-capture-driver-native-jar
			SOURCES
				java/driver/com/github/sarxos/webcam/ds/nativeapi/NativeWebcamDevice.java
				java/driver/com/github/sarxos/webcam/ds/nativeapi/NativeWebcamDriver.java
			INCLUDE_JARS
				${webcam-capture-jar}
			OUTPUT_NAME
				webcam-capture-driver-native
)

create_javah(TARGET webcam-capture-driver-native-javah
				CLASSES
					com.github.sarxos.webcam.ds.nativeapi.NativeWebcamDevice
					com.github.sarxos.webcam.ds.nativeapi.NativeWebcamDriver
				CLASSPATH
					webcam-capture-driver-native-jar;${webcam-capture-jar}
)

add_library(webcam-capture-driver-native-${bitness} SHARED
				jni/com_github_sarxos_webcam_ds_nativeapi_NativeWebcamDevice.cpp
				jni/com_github_sarxos_webcam_ds_nativeapi_NativeWebcamDriver.cpp
)
target_include_directories(webcam-capture-driver-native-${bitness} PRIVATE ${JNI_INCLUDE_DIRS} ${CMAKE_BINARY_DIR} ${video-capture-library-path}/include ${libyuv-path}/include)
add_dependencies(webcam-capture-driver-native-${bitness} webcam-capture-driver-native-javah)
target_link_libraries(webcam-capture-driver-native-${bitness} videocapture yuv)



set(JAVA_CP_SEPARATOR ":")
if(WIN32)
    set(JAVA_CP_SEPARATOR ";")
endif()


add_jar(WebcamCaptureTestUseNativeDriver
			SOURCES
				test/WebcamCaptureTestUseNativeDriver.java
			INCLUDE_JARS
				${webcam-capture-jar}
				webcam-capture-driver-native-jar
)
add_dependencies(WebcamCaptureTestUseNativeDriver webcam-capture-driver-native-${bitness})
add_test(NAME TestWebcamCaptureTestUseNativeDriver COMMAND ${Java_JAVA_EXECUTABLE} -Djava.library.path=$<CONFIGURATION> -cp "WebcamCaptureTestUseNativeDriver.jar${JAVA_CP_SEPARATOR}webcam-capture-driver-native.jar${JAVA_CP_SEPARATOR}${webcam-capture-jar}${JAVA_CP_SEPARATOR}${slf4j-jar}" WebcamCaptureTestUseNativeDriver)

add_executable(WebcamCaptureTestCaptureToFile test/WebcamCaptureTestCaptureToFile.cpp)
target_include_directories(WebcamCaptureTestCaptureToFile PRIVATE ${CMAKE_BINARY_DIR} ${video-capture-library-path}/include ${libyuv-path}/include)
target_link_libraries(WebcamCaptureTestCaptureToFile videocapture yuv)
add_test(NAME TestWebcamCaptureTestCaptureToFile COMMAND WebcamCaptureTestCaptureToFile)

add_jar(WebcamCaptureTestShowCapturedFile
			SOURCES
				test/WebcamCaptureTestShowCapturedFile.java
)
add_dependencies(WebcamCaptureTestShowCapturedFile WebcamCaptureTestCaptureToFile)
add_test(NAME TestWebcamCaptureTestShowCapturedFile COMMAND ${Java_JAVA_EXECUTABLE} -cp "WebcamCaptureTestShowCapturedFile.jar" WebcamCaptureTestShowCapturedFile "captured.rgb")


set(webcam-capture-install-bin-directory ${CMAKE_CURRENT_SOURCE_DIR}/../bin)
install(TARGETS webcam-capture-driver-native-${bitness} RUNTIME DESTINATION ${webcam-capture-install-bin-directory} LIBRARY DESTINATION ${webcam-capture-install-bin-directory})
install_jar(webcam-capture-driver-native-jar ${webcam-capture-install-bin-directory})


# Build with /MT instead of /MD on Windows
if(MSVC)
	foreach(flag_var
			CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
			CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
	   if(${flag_var} MATCHES "/MD")
		  string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
	   endif(${flag_var} MATCHES "/MD")
	endforeach(flag_var)
endif()