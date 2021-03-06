##---------------------------------------------------------------------------
## Author:      Pavel Kalian
## Copyright:   2014
## License:     GPLv2 License
##---------------------------------------------------------------------------

# define minimum cmake version
CMAKE_MINIMUM_REQUIRED(VERSION 3.1.1)
CMAKE_POLICY(SET CMP0043 NEW)

PROJECT(chartdldr_pi)

SET(PACKAGE_NAME chartdldr_pi)
SET(VERBOSE_NAME ChartDownloader)
SET(TITLE_NAME ChartDownloader)
SET(CPACK_PACKAGE_CONTACT "Pavel Kalian")

SET(VERSION_MAJOR "1")
SET(VERSION_MINOR "4")
SET(VERSION_PATCH "0")
SET(VERSION_DATE "2019-12-09")

IF( NOT CMAKE_BUILD_TYPE )
  SET( CMAKE_BUILD_TYPE Release CACHE STRING
       "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
       FORCE )
ENDIF( NOT CMAKE_BUILD_TYPE )

INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/include)

MESSAGE (STATUS "Build type: ${CMAKE_BUILD_TYPE}")

#SET(CMAKE_BUILD_TYPE Debug)
SET(CMAKE_MACOSX_RPATH "ON")

INCLUDE("cmake/PluginConfigure.cmake")

SET(SRC_CHARTDLDR
            src/chartdldr_pi.h
            src/chartdldr_pi.cpp
            src/icons.h
            src/icons.cpp
            src/chartdldrgui.h
            src/chartdldrgui.cpp
            src/checkedlistctrl.cpp
            src/checkedlistctrl.h
            src/base64.h
            src/chartcatalog.cpp
            src/chartcatalog.h
            src/pugiconfig.hpp
            src/pugixml.hpp
            src/pugixml.cpp
)
            
IF(QT_ANDROID)
    SET(SRC_CHARTDLDR ${SRC_CHARTDLDR}  src/androidSupport.h)
    SET(SRC_CHARTDLDR ${SRC_CHARTDLDR} src/androidSupport.cpp)
ENDIF(QT_ANDROID)

SET(SRC_UNARR
    src/unarr/common/conv.c
    src/unarr/common/crc32.c
    src/unarr/common/custalloc.c
    src/unarr/common/allocator.h
    src/unarr/common/unarr.c
    src/unarr/common/unarr-imp.h
    src/unarr/common/stream.c
    src/unarr/_7z/_7z.h
    src/unarr/_7z/_7z.c
    src/unarr/lzmasdk/Ppmd7Dec.c
    src/unarr/lzmasdk/Ppmd8.h
    src/unarr/lzmasdk/7zTypes.h
    src/unarr/lzmasdk/Precomp.h
    src/unarr/lzmasdk/CpuArch.h
    src/unarr/lzmasdk/Ppmd7.c
    src/unarr/lzmasdk/LzmaDec.c
    src/unarr/lzmasdk/Ppmd8.c
    src/unarr/lzmasdk/Ppmd.h
    src/unarr/lzmasdk/Ppmd7.h
    src/unarr/lzmasdk/CpuArch.c
    src/unarr/lzmasdk/LzmaDec.h
    src/unarr/lzmasdk/Ppmd8Dec.c
    src/unarr/rar/filter-rar.c
    src/unarr/rar/huffman-rar.c
    src/unarr/rar/parse-rar.c
    src/unarr/rar/rar.h
    src/unarr/rar/rarvm.c
    src/unarr/rar/uncompress-rar.c
    src/unarr/rar/rarvm.h
    src/unarr/rar/lzss.h
    src/unarr/rar/rar.c
    src/unarr/unarr.h
    src/unarr/tar/tar.h
    src/unarr/tar/parse-tar.c
    src/unarr/tar/tar.c
    src/unarr/zip/uncompress-zip.c
    src/unarr/zip/inflate.c
    src/unarr/zip/zip.c
    src/unarr/zip/parse-zip.c
    src/unarr/zip/zip.h
    src/unarr/zip/inflate.h
)

IF(APPLE)
  #Homebrew does not link libarchive to /usr/local by default to avoid conflict with the system-wide one (which in turn does not provide includes...)
  #We do not want the systemwide vrsion as it is too old and misses support for many archive formats
  LIST(APPEND CMAKE_PREFIX_PATH "/usr/local/opt/libarchive")
  LIST(APPEND CMAKE_PREFIX_PATH "/usr/local/opt/libarchive/lib")
ENDIF()

IF(CMAKE_HOST_WIN32)
  LIST(APPEND CMAKE_PREFIX_PATH "../../buildwin/include")
  LIST(APPEND CMAKE_PREFIX_PATH "../../buildwin")
ENDIF()

OPTION(CHARTDLDR_USE_UNARR "Use unarr to extract RAR archives instead of libarchive" ON)
IF(CHARTDLDR_USE_UNARR)
    ADD_DEFINITIONS(-DCHARTDLDR_RAR_UNARR)
ENDIF()

ADD_DEFINITIONS(-D_FILE_OFFSET_BITS=64)

FIND_PACKAGE(LibArchive)
IF(LibArchive_FOUND)
  message(STATUS "chartdldr_pi: going to use LibArchive...")
  message(STATUS "chartdldr_pi: " ${LibArchive_INCLUDE_DIRS})
  message(STATUS "chartdldr_pi: " ${LibArchive_LIBRARIES})
  INCLUDE_DIRECTORIES(${LibArchive_INCLUDE_DIRS})
  #Unset the unrar sources as we will not need them
  ADD_DEFINITIONS(-DDLDR_USE_LIBARCHIVE)
ENDIF(LibArchive_FOUND)

INCLUDE_DIRECTORIES(src/unarr/)
PKG_SEARCH_MODULE(UNARR libunarr unarr)
USE_BUNDLED_LIB(USE_BUNDLED_UNARR unarr)
IF (UNARR_FOUND AND NOT USE_BUNDLED_UNARR)
  message (STATUS "chartdldr_pi: Building with system unarr")
  INCLUDE_DIRECTORIES(${UNARR_INCLUDE_DIR})
  SET(SRC_UNARR  "")
ELSE ()
  message (STATUS "chartdldr_pi: Building with bundled unarr")
  ADD_LIBRARY(UNARR ${SRC_UNARR})
  INCLUDE_DIRECTORIES(src/unarr)
ENDIF ()

IF(QT_ANDROID)
  ADD_DEFINITIONS(-D__STDC_LIMIT_MACROS)
  ADD_DEFINITIONS(-D__STDC_CONSTANT_MACROS)
  ADD_DEFINITIONS("-Wno-inconsistent-missing-override -Wno-potentially-evaluated-expression")
ENDIF(QT_ANDROID)

ADD_LIBRARY(${PACKAGE_NAME} SHARED ${SRC_UNARR} ${SRC_CHARTDLDR})

if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang|AppleClang")
    # FIXME: This is to broad, needs better handling after 5.0.0
    target_compile_options(${PACKAGE_NAME} PRIVATE -Wno-overloaded-virtual)
    target_compile_options(${PACKAGE_NAME} PRIVATE -Wno-unknown-pragmas)
    target_compile_options(${PACKAGE_NAME} PRIVATE -Wno-unused-variable)
endif ()

IF (UNARR_LIBRARIES)
    TARGET_LINK_LIBRARIES(${PACKAGE_NAME} PRIVATE ${UNARR_LIBRARIES})
ENDIF()

IF(LibArchive_FOUND)
    TARGET_LINK_LIBRARIES(${PACKAGE_NAME} PRIVATE ${LibArchive_LIBRARIES})
ELSE(LibArchive_FOUND)
  #We do not have libarchive, try to make unarr more powerful...
  #ADD_DEFINITIONS(-DHAVE_7Z) #TODO if we decide it is needed, quite a bit more work here
  FIND_PACKAGE(ZLIB)
  IF(ZLIB_FOUND)
    ADD_DEFINITIONS(-DHAVE_ZLIB)
    INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIRS})
    TARGET_LINK_LIBRARIES(${PACKAGE_NAME} PRIVATE ${ZLIB_LIBRARIES})
  ENDIF(ZLIB_FOUND)
  FIND_PACKAGE(BZip2)
  IF(BZIP2_FOUND)
    ADD_DEFINITIONS(-DHAVE_BZIP2)
    INCLUDE_DIRECTORIES(${BZIP2_INCLUDE_DIRS})
    TARGET_LINK_LIBRARIES(${PACKAGE_NAME} PRIVATE ${BZIP2_LIBRARIES})
  ENDIF(BZIP2_FOUND)
ENDIF(LibArchive_FOUND)

#TARGET_LINK_LIBRARIES( ${PACKAGE_NAME} PRIVATE wxjson::wxjson )

#target_link_libraries( ${PACKAGE_NAME} PRIVATE ${OPENCPN_LINK_TARGET} )

IF (NOT QT_ANDROID )
TARGET_LINK_LIBRARIES( ${PACKAGE_NAME} PRIVATE 
    ${wxWidgets_LIBRARIES}
    ocpn::wxjson
    ocpn::wxsvg
    ocpn::opencpn 
)
ELSE (NOT QT_ANDROID )
  TARGET_LINK_LIBRARIES( ${PACKAGE_NAME} PRIVATE 
    ocpn::wxjson 
    ${Qt_Base}/${Qt_Build}/lib/libQt5Gui.so 
    ${Qt_Base}/${Qt_Build}/lib/libQt5Core.so 
    ${Qt_Base}/${Qt_Build}/lib/libQt5Widgets.so
    ${Qt_Base}/${Qt_Build}/lib/libQt5AndroidExtras.so
    ${CMAKE_CURRENT_BINARY_DIR}/../../libgorp.so
    "-lz" 
  )
  
  SET(CMAKE_SHARED_LINKER_FLAGS "-Wl,-soname,lib${PACKAGE_NAME}.so ")
  #SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-s")
ENDIF(NOT QT_ANDROID)

IF( APPLE )
    target_compile_options(${PACKAGE_NAME} PRIVATE
        "-Wall" "-Wno-unused" "-fexceptions"
        "-Wno-overloaded-virtual"
        "-Wno-deprecated" "-Wno-deprecated-declarations"
        "-Wno-unknown-pragmas"
        "-D_WCHAR_H_CPLUSPLUS_98_CONFORMANCE_"
    )
ENDIF(APPLE)

INCLUDE("cmake/PluginInstall.cmake")
INCLUDE("cmake/PluginLocalization.cmake")
INCLUDE("cmake/PluginPackage.cmake")
