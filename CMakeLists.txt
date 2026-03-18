cmake_minimum_required(VERSION 3.16)
project(QuantumPlayer VERSION 2.0.0 LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

# ── Download miniaudio.h if not present ──────────────────────
set(MINIAUDIO_H "${CMAKE_SOURCE_DIR}/third_party/miniaudio.h")
if(NOT EXISTS "${MINIAUDIO_H}")
    message(STATUS "Downloading miniaudio.h...")
    file(DOWNLOAD
        "https://raw.githubusercontent.com/mackron/miniaudio/master/miniaudio.h"
        "${MINIAUDIO_H}"
        STATUS DL_STATUS SHOW_PROGRESS
    )
    list(GET DL_STATUS 0 DL_RESULT)
    if(NOT DL_RESULT EQUAL 0)
        message(FATAL_ERROR
            "Failed to download miniaudio.h!\n"
            "Please run: download_deps.sh (Linux/macOS) or download_deps.bat (Windows)\n"
            "Or manually place miniaudio.h in the third_party/ folder.\n"
            "Download from: https://github.com/mackron/miniaudio/blob/master/miniaudio.h"
        )
    endif()
    message(STATUS "miniaudio.h downloaded OK")
endif()

find_package(Qt6 6.2 REQUIRED COMPONENTS Core Widgets)

qt_add_executable(QuantumPlayer
    src/main.cpp
    src/mainwindow.cpp               src/mainwindow.h

    src/player/audioengine.cpp       src/player/audioengine.h
    src/player/miniaudio_impl.cpp

    src/ui/visualizerwidget.cpp      src/ui/visualizerwidget.h
    src/ui/equalizerwidget.cpp       src/ui/equalizerwidget.h
    src/ui/playlistwidget.cpp        src/ui/playlistwidget.h
    src/ui/librarywidget.cpp         src/ui/librarywidget.h
    src/ui/nowplayingwidget.cpp      src/ui/nowplayingwidget.h
    src/ui/thememanager.cpp          src/ui/thememanager.h
    src/ui/levelmeterswidget.cpp     src/ui/levelmeterswidget.h
    src/ui/sleeptimerwidget.cpp      src/ui/sleeptimerwidget.h
    src/ui/clickableslider.h

    src/library/track.h
    src/library/album.h
    src/library/artist.h
    src/library/musiclibrary.cpp     src/library/musiclibrary.h

    src/resources/resources.qrc
)

target_include_directories(QuantumPlayer PRIVATE
    src
    third_party
)

target_link_libraries(QuantumPlayer PRIVATE
    Qt6::Core
    Qt6::Widgets
)

# Platform-specific audio libs needed by miniaudio
if(WIN32)
    target_link_libraries(QuantumPlayer PRIVATE
        ole32 user32 advapi32
    )
elseif(APPLE)
    find_library(COREAUDIO CoreAudio REQUIRED)
    find_library(AUDIOTOOLBOX AudioToolbox REQUIRED)
    find_library(COREFOUNDATION CoreFoundation REQUIRED)
    target_link_libraries(QuantumPlayer PRIVATE
        ${COREAUDIO} ${AUDIOTOOLBOX} ${COREFOUNDATION}
    )
else()
    # Linux — need ALSA or PulseAudio
    find_package(PkgConfig QUIET)
    if(PKG_CONFIG_FOUND)
        pkg_check_modules(ALSA QUIET alsa)
        pkg_check_modules(PULSE QUIET libpulse)
    endif()
    if(ALSA_FOUND)
        target_link_libraries(QuantumPlayer PRIVATE ${ALSA_LIBRARIES})
        target_compile_definitions(QuantumPlayer PRIVATE MA_ENABLE_ALSA)
    endif()
    if(PULSE_FOUND)
        target_link_libraries(QuantumPlayer PRIVATE ${PULSE_LIBRARIES})
    endif()
    target_link_libraries(QuantumPlayer PRIVATE pthread dl m)
endif()

set_target_properties(QuantumPlayer PROPERTIES
    MACOSX_BUNDLE TRUE
    WIN32_EXECUTABLE TRUE
)

include(GNUInstallDirs)
install(TARGETS QuantumPlayer
    BUNDLE  DESTINATION .
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
