#------------------------------------------------------------
# Copyright (c) 2018 Thomas Kais
#
# This file is subject to the terms and conditions defined in
# file 'COPYING', which is part of this source code package.
#------------------------------------------------------------

# http://doc.qt.io/qt-5/cmake-manual.html
cmake_minimum_required(VERSION 3.9.0 FATAL_ERROR)

project(FotoBox)

# MacOS Qt5 Homebrew fix
if(APPLE AND EXISTS /usr/local/opt/qt5)
    # Homebrew installs Qt5 (up to at least 5.9.1) in /usr/local/qt5, ensure
    # it can be found by CMake since it is not in the default /usr/local prefix.
    list(APPEND CMAKE_PREFIX_PATH "/usr/local/opt/qt5")
endif()

# By default, QT contains core and gui
find_package(Qt5 COMPONENTS Widgets LinguistTools REQUIRED)
if(Qt5Core_FOUND)
    # Ubuntu Xenial Qt v5.5.1
    if(Qt5Core_VERSION VERSION_LESS 5.5.0)
        message(WARNING "Your Qt 5 version '${Qt5Core_VERSION}' is might not be compatible anymore.")
    endif()
endif()

# The following define makes your compiler emit warnings if you use
# any Qt feature that has been marked deprecated (the exact warnings
# depend on your compiler). Please consult the documentation of the
# deprecated API in order to know how to port your code away from it.
add_definitions(-DQT_DEPRECATED_WARNINGS)

# You can also make your code fail to compile if it uses deprecated APIs.
# In order to do so, uncomment the following line.
# You can also select to disable deprecated APIs only up to a certain version of Qt.
add_definitions(-DQT_DISABLE_DEPRECATED_BEFORE=0x060000)    # disables all the APIs deprecated before Qt 6.0.0

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Running moc/uic/rcc automatically
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOUIC_SEARCH_PATHS ${CMAKE_SOURCE_DIR}/forms)

# Find includes in corresponding build directories
set(CMAKE_INCLUDE_CURRENT_DIR ON)
include_directories(${PROJECT_SOURCE_DIR}/include)
set(HEADERS
    ${PROJECT_SOURCE_DIR}/include/buzzer.h
    ${PROJECT_SOURCE_DIR}/include/camera.h
    ${PROJECT_SOURCE_DIR}/include/countdown.h
    ${PROJECT_SOURCE_DIR}/include/fotobox.h
    ${PROJECT_SOURCE_DIR}/include/preferenceprovider.h
    ${PROJECT_SOURCE_DIR}/include/preferences.h
    )
set(SOURCES
    ${PROJECT_SOURCE_DIR}/source/main.cpp
    ${PROJECT_SOURCE_DIR}/source/buzzer.cpp
    ${PROJECT_SOURCE_DIR}/source/camera.cpp
    ${PROJECT_SOURCE_DIR}/source/countdown.cpp
    ${PROJECT_SOURCE_DIR}/source/fotobox.cpp
    ${PROJECT_SOURCE_DIR}/source/preferenceprovider.cpp
    ${PROJECT_SOURCE_DIR}/source/preferences.cpp
    )
set(FORMS
    ${PROJECT_SOURCE_DIR}/forms/fotobox.ui
    ${PROJECT_SOURCE_DIR}/forms/preferences.ui
    ${PROJECT_SOURCE_DIR}/forms/commandlineoptions.ui
    )
set(TS_FILES
    ${PROJECT_SOURCE_DIR}/resources/translation_de.ts
    ${PROJECT_SOURCE_DIR}/resources/translation_en.ts
    )
set(RESOURCES
    ${PROJECT_SOURCE_DIR}/resources/qresource.qrc
    )
set(OTHER
    ${PROJECT_SOURCE_DIR}/.gitignore
    ${PROJECT_SOURCE_DIR}/.travis.yml
    ${PROJECT_SOURCE_DIR}/fotobox.pro
    ${PROJECT_SOURCE_DIR}/COPYING
    ${PROJECT_SOURCE_DIR}/README.md
    ${PROJECT_SOURCE_DIR}/other/DockerfileStretch
    ${PROJECT_SOURCE_DIR}/other/DockerfileBuster
    ${PROJECT_SOURCE_DIR}/other/Doxyfile
    ${PROJECT_SOURCE_DIR}/other/Info.plist
    ${PROJECT_SOURCE_DIR}/other/install_dependencies.sh
    ${PROJECT_SOURCE_DIR}/other/RaspPi_2B_default_GPIO.jpg
    )

# Translation: build qm files in source directory
set_source_files_properties(${TS_FILES} PROPERTIES OUTPUT_LOCATION ${PROJECT_SOURCE_DIR}/resources)
qt5_add_translation(QM_FILES ${TS_FILES})

# WORKAROUND QTBUG-36714: closing a full screen QMainWindow leaves the screen black on macOS if there are multiple instances of QMainWindow created
if(UNIX AND APPLE)
  set(SOURCES_MAC ${PROJECT_SOURCE_DIR}/source/fotoboxmac.mm)
endif()

add_executable(${CMAKE_PROJECT_NAME}
    ${HEADERS}
    ${SOURCES}
    ${SOURCES_MAC}
    ${FORMS}
    ${RESOURCES}
    ${QM_FILES}
    ${OTHER}
    )

# Libraries
target_link_libraries(${CMAKE_PROJECT_NAME} Qt5::Widgets)
if(UNIX AND NOT APPLE)
    # Speed-Up compiling time with ccache (apt-get install ccache)
    find_program(CCACHE_PROGRAM ccache)
    if(CCACHE_PROGRAM)
        set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
        set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK "${CCACHE_PROGRAM}")
    endif()

    # Raspberry Pi wiringPi framework
    list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)
    find_package(WiringPi)
    if(WiringPi_FOUND)
        target_include_directories(${WiringPi_INCLUDE_DIR})
        target_link_libraries(${CMAKE_PROJECT_NAME} optimized ${WiringPi_LIBRARY_REL})
        target_link_libraries(${CMAKE_PROJECT_NAME} debug     ${WiringPi_LIBRARY_DEB})
    endif()
endif()

# WORKAROUND QTBUG-36714: closing a full screen QMainWindow leaves the screen black on macOS if there are multiple instances of QMainWindow created
if(UNIX AND APPLE)
  target_link_libraries(${CMAKE_PROJECT_NAME} "-framework Foundation" "-framework AppKit")
endif()

# Disable the Dark Mode in the app if macOS SDK is Mojave (10.14) and Qt version is less than 5.12
if(UNIX AND APPLE)
  # macOS SDK version
  execute_process(COMMAND xcrun --sdk macosx --show-sdk-version OUTPUT_VARIABLE MAC_SDK_INFO)
  if(MAC_SDK_INFO STREQUAL "")
    message(FATAL_ERROR "Could not resolve SDK" ${MAC_SDK_INFO})
  endif()
  if(MAC_SDK_INFO VERSION_GREATER 10.13)
    if(Qt5Core_VERSION VERSION_LESS 5.12.0)
      # https://developer.apple.com/documentation/appkit/nsappearancecustomization/choosing_a_specific_appearance_for_your_app
      # use Info.plist with NSRequiresAquaSystemAppearance=true to disable Dark Mode
      set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES MACOSX_BUNDLE TRUE MACOSX_BUNDLE_INFO_PLIST ${PROJECT_SOURCE_DIR}/other/Info.plist)
    endif()
  endif()
endif()

# clang tidy
if(ENABLE_CLANG_TIDY)
    find_program(CLANG_TIDY_BIN clang-tidy-7)
    find_program(RUN_CLANG_TIDY_BIN run-clang-tidy-7.py)

    if(CLANG_TIDY_BIN STREQUAL "CLANG_TIDY_BIN-NOTFOUND")
        message(FATAL_ERROR "unable to locate 'clang-tidy-7'")
    endif()

    if(RUN_CLANG_TIDY_BIN STREQUAL "RUN_CLANG_TIDY_BIN-NOTFOUND")
        message(FATAL_ERROR "unable to locate 'run-clang-tidy-7.py'")
    endif()

    # https://releases.llvm.org/7.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html
    list(APPEND RUN_CLANG_TIDY_BIN_ARGS
        -clang-tidy-binary ${CLANG_TIDY_BIN}
        -header-filter='/fotobox/include/'
        # removed checks: cppcoreguidelines-owning-memory / fuchsia-default-arguments
        -checks=-*,android-*,boost-*,bugprone-*,cert-*,clang-analyzer-*,cppcoreguidelines-avoid-goto,cppcoreguidelines-c-copy-assignment-signature,cppcoreguidelines-interfaces-global-init,cppcoreguidelines-narrowing-conversions,cppcoreguidelines-no-malloc,cppcoreguidelines-pro-*,cppcoreguidelines-slicing,cppcoreguidelines-special-member-functions,fuchsia-header-anon-namespaces,fuchsia-multiple-inheritance,fuchsia-overloaded-operator,fuchsia-restrict-system-includes,fuchsia-statically-constructed-objects,fuchsia-trailing-return,fuchsia-virtual-inheritance,google-*,hicpp-*,llvm-*,misc-*,modernize-*,mpi-*,objc-*,performance-*,readability-*
        -extra-arg='-Wno-everything'
    )

    add_custom_target(
        tidy
        COMMAND ${RUN_CLANG_TIDY_BIN} ${RUN_CLANG_TIDY_BIN_ARGS}
        COMMENT "running clang tidy"
    )
endif()

# clang clazy
if(ENABLE_CLANG_CLAZY)
    find_program(CLANG_CLAZY_BIN clazy.AppImage PATH ${CMAKE_CURRENT_SOURCE_DIR} NO_DEFAULT_PATH)

    if(CLANG_CLAZY_BIN STREQUAL "CLANG_CLAZY_BIN-NOTFOUND")
        message(FATAL_ERROR "unable to locate 'clazy.AppImage'")
    endif()

    # https://github.com/KDE/clazy#clazy-standalone-and-json-database-support
    list(APPEND CLANG_CLAZY_BIN_ARGS
        --standalone
        -header-filter='/fotobox/include/'
        -checks='level0,level1,level2'
        -extra-arg='-Wno-everything'
        -ignore-dirs='_autogen'
        -only-qt
        -p='compile_commands.js'
        source/*.cpp
    )

    add_custom_target(
        clazy
        COMMAND ${CLANG_CLAZY_BIN} ${CLANG_CLAZY_BIN_ARGS}
        COMMENT "running clang clazy"
    )
endif()
