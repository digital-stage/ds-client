cmake_minimum_required(VERSION 3.16.0)

if (APPLE)
    include(FixAppleArchitecture.cmake)
endif (APPLE)
#set(CMAKE_OSX_ARCHITECTURES x86_64;arm64)
set(CMAKE_OSX_DEPLOYMENT_TARGET 10.13)


## GLOBAL CONFIGURATION
project(DigitalStage LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_PROJECT_VERSION_MAJOR "0")
set(CMAKE_PROJECT_VERSION_MINOR "1")
set(CMAKE_PROJECT_VERSION_PATCH "0")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -gdwarf-3")
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -gdwarf-3")
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/modules)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)


### RUNTIME FLAGS
add_compile_definitions(GUI)
add_compile_definitions(STAGE_URL="https://single.dstage.org")
add_compile_definitions(AUTH_SERVER="https://single.dstage.org/api/auth")
add_compile_definitions(API_SERVER="wss://single.dstage.org/api/endpoint/")
if (APPLE)
    add_compile_definitions(ZITAPATH="./")
    # Required by AppStore
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -gdwarf-3")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -gdwarf-3")
endIf (APPLE)


### SOURCE AND RESOURCE MANAGEMENT
# Language generation
set(TS_FILES
        i18n/DigitalStage_de.ts
        i18n/DigitalStage_fr.ts
        i18n/DigitalStage_es.ts
        )
# Source files
set(PROJECT_SOURCES
        ${PROJECT_SOURCE_DIR}/src/main.cpp
        ${PROJECT_SOURCE_DIR}/src/app.cpp
        ${PROJECT_SOURCE_DIR}/src/app.h
        ${PROJECT_SOURCE_DIR}/src/logindialog.cpp
        ${PROJECT_SOURCE_DIR}/src/logindialog.h
        ${PROJECT_SOURCE_DIR}/src/api/auth.cpp
        ${PROJECT_SOURCE_DIR}/src/api/auth.h
        ${PROJECT_SOURCE_DIR}/src/api/keystore.cpp
        ${PROJECT_SOURCE_DIR}/src/api/keystore.h
        ${PROJECT_SOURCE_DIR}/src/ov/ov_worker.cpp
        ${PROJECT_SOURCE_DIR}/src/ov/ov_worker.h
        ${PROJECT_SOURCE_DIR}/src/ov/ov_controller.cpp
        ${PROJECT_SOURCE_DIR}/src/ov/ov_controller.h
        ${PROJECT_SOURCE_DIR}/src/helper/filedownloader.cpp
        ${PROJECT_SOURCE_DIR}/src/helper/filedownloader.h
        resources.qrc
        )
file(GLOB
        PROJECT_ASSETS
        ${CMAKE_CURRENT_SOURCE_DIR}/i18n/*.qm
        ${CMAKE_CURRENT_SOURCE_DIR}/images/*.png
        ${CMAKE_CURRENT_SOURCE_DIR}/images/*.svg
        )

### DEPENDENCIES
if (APPLE)
    include(BrewResolver.cmake)
endif(APPLE)
if (UNIX AND NOT APPLE)
    if (EXISTS "/usr/lib/x86_64-linux-gnu/cmake")
        message("FIX: Resolving a bug produced by malformed cpprestsdk packages, see: https://bugs.launchpad.net/ubuntu/+source/cpprest/+bug/1838826")
        set(CMAKE_PREFIX_PATH "${CMAKE_PREFIX_PATH};/usr/lib/x86_64-linux-gnu/cmake")
    endif ()
    if (EXISTS "/usr/lib/arm-linux-gnueabihf/cmake")
        message("FIX: Resolving a bug produced by malformed cpprestsdk packages, see: https://bugs.launchpad.net/ubuntu/+source/cpprest/+bug/1838826")
        set(CMAKE_PREFIX_PATH "${CMAKE_PREFIX_PATH};/usr/lib/arm-linux-gnueabihf/cmake")
    endif ()
endif ()

if(USE_CONAN)
    if(NOT EXISTS "${CMAKE_BINARY_DIR}/conan.cmake")
        message(STATUS "Downloading conan.cmake from https://github.com/conan-io/cmake-conan")
        file(DOWNLOAD "https://raw.githubusercontent.com/conan-io/cmake-conan/v0.16.1/conan.cmake"
                    "${CMAKE_BINARY_DIR}/conan.cmake"
                    TLS_VERIFY ON)
    endif()
    include(${CMAKE_BINARY_DIR}/conan.cmake)

    #conan_cmake_autodetect(settings)

    conan_cmake_run(
    #    SETTINGS os=Macos
    #    SETTINGS os_build=Macos
    #    SETTINGS arch=x86_64
    #    SETTINGS arch_build=x86_64
    #    SETTINGS compiler=apple-clang
        SETTINGS compiler.version=9.1)

    conan_cmake_configure(REQUIRES fftw/3.3.9
                      GENERATORS cmake_find_package)
    #conan_cmake_configure(REQUIRES xerces-c/3.2.3
    #                GENERATORS cmake_find_package)
    #conan_cmake_configure(REQUIRES cpprestsdk/2.10.18
    #                GENERATORS cmake_find_package)
    #conan_cmake_configure(REQUIRES openssl/1.1.1j
    #                GENERATORS cmake_find_package)

    message(${settings})

    conan_cmake_install(PATH_OR_REFERENCE .
                        BUILD missing
                        REMOTE conan-center
                        SETTINGS ${settings})
endif(USE_CONAN)


find_package(QT NAMES Qt5 COMPONENTS Widgets Network LinguistTools REQUIRED)
find_package(Qt${QT_VERSION_MAJOR} COMPONENTS Widgets Network LinguistTools REQUIRED)
message("Using Qt${QT_VERSION_MAJOR}")
if (UNIX AND NOT APPLE)
    find_package(ALSA REQUIRED)
endif ()
find_package(Qt5Keychain REQUIRED)
find_package(cpprestsdk REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(XercesC REQUIRED)
find_package(Jack REQUIRED)


# libov
add_subdirectory(libov EXCLUDE_FROM_ALL)
# Tascar plugins
set(PLUGINPREFIX "ovclient")
list(APPEND RECEIVERS
        ortf
        hrtf
        simplefdn
        omni
        )
list(APPEND TASCARMODS
        system
        touchosc
        waitforjackport
        route
        jackrec
        )
list(APPEND SOURCEMODS
        omni
        cardioidmod
        )
list(APPEND AUDIOPLUGINS
        sndfile
        delay
        metronome
        )
add_subdirectory(libov/tascar/plugins ${PROJECT_BINARY_DIR}/plugins EXCLUDE_FROM_ALL)
get_property(TASCAR_PLUGIN_LIBRARIES GLOBAL PROPERTY TASCAR_PLUGIN_LIBRARIES)
# zita-njbridge executables
add_subdirectory(zita-njbridge ${PROJECT_BINARY_DIR}/zita EXCLUDE_FROM_ALL)


### TARGETS
if (APPLE)
    # Add app icon
    set(APP_ICON ${CMAKE_CURRENT_SOURCE_DIR}/images/appicon.icns)
    set_source_files_properties(${APP_ICON} PROPERTIES
            MACOSX_PACKAGE_LOCATION "Resources")
endif ()

if (${QT_VERSION_MAJOR} GREATER_EQUAL 6)
    qt_add_executable(DigitalStage ${PROJECT_SOURCES} ${APP_ICON})
    qt_create_translation(QM_FILES ${CMAKE_SOURCE_DIR} ${TS_FILES})
else ()
    add_executable(DigitalStage ${PROJECT_SOURCES} ${APP_ICON})
    qt5_create_translation(QM_FILES ${CMAKE_SOURCE_DIR} ${TS_FILES})
endif ()

target_include_directories(DigitalStage
        PRIVATE
        libov/src
        ${JACK_INCLUDE_DIRS}
        )
target_link_libraries(DigitalStage
        PRIVATE
        Qt${QT_VERSION_MAJOR}::Widgets
        Qt${QT_VERSION_MAJOR}::Network
        qt5keychain
        cpprestsdk::cpprest
        OpenSSL::Crypto
        ${JACK_LIBRARIES}
        #${Boost_SYSTEM_LIBRARY}
        ov
        PUBLIC
        ${TASCAR_PLUGIN_LIBRARIES}
        )
if (UNIX AND NOT APPLE)
    target_link_libraries(DigitalStage
            PRIVATE
            ALSA::ALSA
            )
    set_target_properties(DigitalStage PROPERTIES OUTPUT_NAME "ds-client")
endif ()

### CREATE BUNDLE
if (APPLE)
    set_target_properties(DigitalStage PROPERTIES
            MACOSX_BUNDLE TRUE
            MACOSX_BUNDLE_NAME DigitalStage
            MACOSX_BUNDLE_BUNDLE_NAME DigitalStage
            MACOSX_BUNDLE_GUI_IDENTIFIER org.digital-stage.client
            MACOSX_BUNDLE_BUNDLE_VERSION ${CMAKE_PROJECT_VERSION_MAJOR}.${CMAKE_PROJECT_VERSION_MINOR}.${CMAKE_PROJECT_VERSION_PATCH}
            MACOSX_BUNDLE_ICON_FILE appicon.icns
            MACOSX_BUNDLE_COPYRIGHT GPLv2
            MACOSX_BUNDLE_INFO_STRING "Digital Stage Client"
            MACOSX_FRAMEWORK_IDENTIFIER org.digital-stage.client
            )
    if (FALSE)
            add_custom_command(TARGET DigitalStage
                    POST_BUILD COMMAND
                    ${CMAKE_INSTALL_NAME_TOOL} -add_rpath "@executable_path/../../Frameworks/"
                    $<TARGET_FILE:DigitalStage>)
    endif ()
endif()
# Copy zita binaries into the application bundle
add_custom_command(TARGET DigitalStage
    POST_BUILD
    COMMAND cp $<TARGET_FILE_DIR:zita-n2j>/zita-n2j
    $<TARGET_FILE_DIR:DigitalStage>
    COMMENT "Copying zita-n2j" VERBATIM)
add_custom_command(TARGET DigitalStage
    POST_BUILD
    COMMAND cp $<TARGET_FILE_DIR:zita-j2n>/zita-j2n
    $<TARGET_FILE_DIR:DigitalStage>
    COMMENT "Copying zita-j2n" VERBATIM)


## INSTALL
install(
        TARGETS DigitalStage
        EXPORT "${TARGETS_EXPORT_NAME}"
        BUNDLE DESTINATION . COMPONENT Runtime
        RUNTIME DESTINATION bin COMPONENT Runtime
        #BUNDLE DESTINATION "/Applications"
        #LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        #ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        #RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
        #INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
        #PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}"
)


## CPACK
if (CMAKE_BUILD_TYPE MATCHES Release)

    set(CPACK_PACKAGE_HOMEPAGE_URL "https://www.digital-stage.org")
    set(CPACK_PACKAGE_CONTACT "Tobias Hegemann <tobias.hegemann@digital-stage.org")

    if (APPLE)
        find_program(MACDEPLOYQT_EXECUTABLE macdeployqt HINTS "${_qt_bin_dir}")

        # First use macdeployqt to bundle the qt libs, but do NOT sign yet
        add_custom_command(TARGET DigitalStage POST_BUILD
                COMMAND "${MACDEPLOYQT_EXECUTABLE}"
                "$<TARGET_FILE_DIR:DigitalStage>/../.."
                -libpath="${PROJECT_BINARY_DIR}/plugins"
                -appstore-compliant
                -always-overwrite
                COMMENT "Running macdeployqt..."
                )
        # We have to fix the dynlink paths of the zita-n2j and zita-j2n, too
        add_custom_command(TARGET DigitalStage POST_BUILD
                COMMAND
                ${CMAKE_INSTALL_NAME_TOOL}
                -change
                "/usr/local/lib/libjack.0.1.0.dylib"
                "@executable_path/../Frameworks/libjack.0.1.0.dylib"
                "$<TARGET_FILE_DIR:DigitalStage>/zita-n2j"
                COMMENT "Fixing zita-n2j paths..."
                )
        add_custom_command(TARGET DigitalStage POST_BUILD
                COMMAND
                ${CMAKE_INSTALL_NAME_TOOL}
                -change
                "/usr/local/lib/libjack.0.1.0.dylib"
                "@executable_path/../Frameworks/libjack.0.1.0.dylib"
                "$<TARGET_FILE_DIR:DigitalStage>/zita-j2n"
                COMMENT "Fixing zita-j2n paths..."
                )
        # Now we have to fix the dynlib paths to the tascar plugins as well
        foreach (TASCAR_PLUGIN IN LISTS TASCAR_PLUGIN_LIBRARIES)
            if (FALSE)
                #This is ommited, since macdeployqt seems to copy the tascar libraries, but is NOT applying the new paths, so...
                add_custom_command(TARGET DigitalStage POST_BUILD
                        COMMAND
                        COMMAND cp "${PROJECT_BINARY_DIR}/plugins/${TASCAR_PLUGIN}.dylib"
                        "$<TARGET_FILE_DIR:DigitalStage>/../Frameworks/${TASCAR_PLUGIN}.dylib"
                        COMMENT "Copy tascar plugins..."
                        )
            endif ()
            # Chaning the dynlib signature of the tascar plugin
            add_custom_command(TARGET DigitalStage POST_BUILD
                    COMMAND
                    ${CMAKE_INSTALL_NAME_TOOL}
                    -change
                    "@rpath/${TASCAR_PLUGIN}.dylib"
                    "@executable_path/../Frameworks/${TASCAR_PLUGIN}.dylib"
                    $<TARGET_FILE:DigitalStage>
                    COMMENT "Fixing tascar paths..."
                    )
        endforeach ()
        # Now signing the app bundle
        set(APPLE_CERT "" CACHE STRING "The name of your Apple supplied code signing certificate for the application. The name usually takes the form Developer ID Application: [Name] or 3rd Party Mac Developer Application: [Name].")
        if (NOT APPLE_CERT STREQUAL "")
            set(ENV{CODESIGN_ALLOCATE} /Applications/Xcode.app/Contents/Developer/usr/bin/codesign_allocate)
            add_custom_command(TARGET DigitalStage POST_BUILD
                    COMMAND
                    codesign
                    --force
                    --deep
                    --sign "${APPLE_CERT}"
                    "${PROJECT_BINARY_DIR}/DigitalStage.app"
                    COMMENT "Signing app bundle using certificate ${APPLE_CERT}..."
                    VERBATIM
                    )
        elseif ()
            message(WARNING "Not signing app bundle, since no APPLE_CERT is specified")
        endif ()

        install(DIRECTORY "${PROJECT_BINARY_DIR}/DigitalStage.app"
                DESTINATION "."
                USE_SOURCE_PERMISSIONS)

        set(CPACK_GENERATOR "DragNDrop")
        set(CPACK_DMG_VOLUME_NAME "DigitalStageInstaller")
        set(CPACK_DMG_DS_STORE_SETUP_SCRIPT ${CMAKE_CURRENT_SOURCE_DIR}/Packaging_DMGSetup.scpt)
        set(CPACK_DMG_BACKGROUND_IMAGE ${CMAKE_CURRENT_SOURCE_DIR}/images/DMGBackground.tif)
        set(CPACK_SOURCE_STRIP_FILES "")
        #set(CPACK_COMPONENTS_ALL "${CMAKE_BINARY_DIR}/DigitalStage.app")
    endif ()

    if (UNIX AND NOT APPLE)
        set(CPACK_PACKAGE_NAME "ds-client")
        set(CPACK_GENERATOR "DEB")
        set(CPACK_DEBIAN_PACKAGE_DEPENDS "libtascar (>= 0.6)")
    endif ()

    include(CPack)
else ()
    message(WARNING "Not isolating app bundle, set CMAKE_BUILD_TYPE to Release to fix the app bundle and finish deploy preparation.")
endif ()
