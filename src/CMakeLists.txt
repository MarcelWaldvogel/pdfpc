find_package(PkgConfig)
pkg_check_modules(GOBJECT REQUIRED gobject-2.0)
pkg_check_modules(GIO REQUIRED gio-2.0)
pkg_check_modules(GEE REQUIRED gee-0.8)
pkg_check_modules(POPPLER REQUIRED poppler-glib>=0.22)
pkg_check_modules(GTK REQUIRED gtk+-3.0>=3.10)
list (FIND GTK_STATIC_LIBRARIES "X11" _index)
if (${_index} GREATER -1)
    set(WITH_X11 1)
    message(STATUS "GTK is compiled with X11 -- enabling X11 support")
    pkg_check_modules(X11 REQUIRED x11)
    set(EXTRA_VALA_OPTIONS ${EXTRA_VALA_OPTIONS} -D X11)
    set(X11_PACKAGE x11)
endif ()
pkg_check_modules(GTHREAD REQUIRED gthread-2.0)
pkg_check_modules(PANGOCAIRO REQUIRED pangocairo)
if (MOVIES)
    pkg_check_modules(GSTREAMER REQUIRED gstreamer-1.0)
    pkg_check_modules(GSTBASE REQUIRED gstreamer-base-1.0)
    pkg_check_modules(GSTAUDIO REQUIRED gstreamer-audio-1.0)
    pkg_check_modules(GSTVIDEO REQUIRED gstreamer-video-1.0)
    set(MOVIE_PACKAGES
        gstreamer-1.0
        gstreamer-base-1.0
        gstreamer-audio-1.0
        gstreamer-video-1.0
    )
endif ()

if ("${GTK_VERSION}" VERSION_LESS 3.18 OR "${VALA_VERSION}" VERSION_LESS 0.30)
    message(WARNING "Using legacy GTK API")
    set(EXTRA_VALA_OPTIONS ${EXTRA_VALA_OPTIONS} -D GTK_LEGACY)
endif ()

if ("${POPPLER_VERSION}" VERSION_GREATER 0.53 AND "${VALA_VERSION}" VERSION_GREATER 0.36.3)
    set(EXTRA_VALA_OPTIONS ${EXTRA_VALA_OPTIONS} -D NEW_POPPLER)
endif()

include_directories(
    ${GOBJECT_INCLUDE_DIRS}
    ${GIO_INCLUDE_DIRS}
    ${GEE_INCLUDE_DIRS}
    ${POPPLER_INCLUDE_DIRS}
    ${GTK_INCLUDE_DIRS}
    ${GTHREAD_INCLUDE_DIRS}
    ${PANGOCAIRO_INCLUDE_DIRS}
    ${GSTREAMER_INCLUDE_DIRS}
    ${GSTINTERFACES_INCLUDE_DIRS}
    ${GSTVIDEO_INCLUDE_DIRS}
    ${CMAKE_CURRENT_SOURCE_DIR}
)

link_directories(
    ${GOBJECT_LIBRARY_DIRS}
    ${GIO_LIBRARY_DIRS}
    ${GEE_LIBRARY_DIRS}
    ${POPPLER_LIBRARY_DIRS}
    ${GTK_LIBRARY_DIRS}
    ${GTHREAD_LIBRARY_DIRS}
    ${PANGOCAIRO_LIBRARY_DIRS}
    ${GSTREAMER_LIBRARY_DIRS}
    ${GSTINTERFACES_LIBRARY_DIRS}
    ${GSTVIDEO_LIBRARY_DIRS}
)

if(${WITH_X11})
    include_directories(${X11_INCLUDE_DIRS})
    link_directories(${X11_LIBRARY_DIRS})
    set(ADDITIONAL_LIBS ${X11_LIBRARIES})
endif()

CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/paths.in ${CMAKE_CURRENT_BINARY_DIR}/paths.vala)

file (GLOB_RECURSE C_SRC RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} *.c)
file (GLOB_RECURSE VALA_SRC RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} *.vala)
list(REMOVE_ITEM VALA_SRC paths.vala)
if (MOVIES)
    set(EXTRA_VALA_OPTIONS ${EXTRA_VALA_OPTIONS} -D MOVIES)
else ()
    LIST(REMOVE_ITEM VALA_SRC classes/action/movie.vala)
endif ()

vala_precompile(VALA_C
    ${VALA_SRC}
PACKAGES
    gio-2.0
    gee-0.8
    poppler-glib
    ${X11_PACKAGE}
    gtk+-3.0
    pangocairo
    posix
    ${MOVIE_PACKAGES}
OPTIONS
    --thread
    --debug
    --enable-experimental
    ${EXTRA_VALA_OPTIONS}
GENERATE_HEADER
    presenter
GENERATE_VAPI
    presenter
CUSTOM_VAPIS
    ${CMAKE_CURRENT_BINARY_DIR}/paths.vala
    ${CMAKE_CURRENT_SOURCE_DIR}/custom_binding.vapi
)

add_executable(pdfpc
    ${VALA_C}
    ${C_SRC}
)

# explicitly add libraries (needed e.g. for Fedora 13+)
target_link_libraries(pdfpc
    m
    ${GOBJECT_LIBRARIES}
    ${GIO_LIBRARIES}
    ${GEE_LIBRARIES}
    ${POPPLER_LIBRARIES}
    ${GTK_LIBRARIES}
    ${GTHREAD_LIBRARIES}
    ${PANGOCAIRO_LIBRARIES}
    ${GSTREAMER_LIBRARIES}
    ${GSTINTERFACES_LIBRARIES}
    ${GSTVIDEO_LIBRARIES}
    ${ADDITIONAL_LIBS}
)

install(TARGETS
    pdfpc
RUNTIME
DESTINATION
    bin
)

# The tests need a library version of all the pdf_presenter_stuff
add_library(pdf_presenter_console_test_library
    SHARED
    EXCLUDE_FROM_ALL
    ${VALA_C} presenter.h presenter.vapi
)
