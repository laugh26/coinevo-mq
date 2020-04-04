cmake_minimum_required(VERSION 3.7)

project(libcoinevomq CXX)

include(GNUInstallDirs)

set(COINEVOMQ_VERSION_MAJOR 1)
set(COINEVOMQ_VERSION_MINOR 0)
set(COINEVOMQ_VERSION_PATCH 4)
set(COINEVOMQ_VERSION "${COINEVOMQ_VERSION_MAJOR}.${COINEVOMQ_VERSION_MINOR}.${COINEVOMQ_VERSION_PATCH}")
message(STATUS "coinevomq v${COINEVOMQ_VERSION}")

set(COINEVOMQ_LIBVERSION 0)


option(BUILD_SHARED_LIBS "Build shared libraries instead of static ones" ON)


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

configure_file(coinevomq/version.h.in coinevomq/version.h @ONLY)
configure_file(libcoinevomq.pc.in libcoinevomq.pc @ONLY)

add_library(coinevomq
    coinevomq/auth.cpp
    coinevomq/bt_serialize.cpp
    coinevomq/connections.cpp
    coinevomq/jobs.cpp
    coinevomq/coinevomq.cpp
    coinevomq/proxy.cpp
    coinevomq/worker.cpp
)
set_target_properties(coinevomq PROPERTIES SOVERSION ${COINEVOMQ_LIBVERSION})

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
target_link_libraries(coinevomq PRIVATE Threads::Threads)

# libzmq is nearly impossible to link statically from a system-installed static library: it depends
# on a ton of other libraries, some of which are not all statically available.  If the caller wants
# to mess with this, so be it: they can set up a libzmq target and we'll use it.  Otherwise if they
# asked us to do things statically, don't even try to find a system lib and just build it.
set(coinevomq_build_static_libzmq OFF)
if(TARGET libzmq)
    target_link_libraries(coinevomq PUBLIC libzmq)
elseif(BUILD_SHARED_LIBS)
    include(FindPkgConfig)
    pkg_check_modules(libzmq libzmq>=4.3 IMPORTED_TARGET)
    if(libzmq_FOUND)
        target_link_libraries(coinevomq PUBLIC PkgConfig::libzmq)
    else()
        set(coinevomq_build_static_libzmq ON)
    endif()
else()
    set(coinevomq_build_static_libzmq ON)
endif()

if(coinevomq_build_static_libzmq)
    message(STATUS "libzmq >= 4.3 not found or static build requested, building bundled 4.3.2")
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/local-libzmq")
    include(LocalLibzmq)
    target_link_libraries(coinevomq PUBLIC libzmq_vendor)
endif()

target_include_directories(coinevomq
    PUBLIC
        $<INSTALL_INTERFACE:>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/cppzmq>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/mapbox-variant/include>
)

target_compile_options(coinevomq PRIVATE -Wall -Wextra -Werror)
set_target_properties(coinevomq PROPERTIES
    CXX_STANDARD 14
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
    POSITION_INDEPENDENT_CODE ON
)

function(link_dep_libs target linktype libdirs)
    foreach(lib ${ARGN})
        find_library(link_lib-${lib} NAMES ${lib} PATHS ${libdirs})
        message(STATUS "FIND ${lib} FOUND ${link_lib-${lib}}")
        if(link_lib-${lib})
            target_link_libraries(${target} ${linktype} ${link_lib-${lib}})
        endif()
    endforeach()
endfunction()

# If the caller has already set up a sodium target then we will just link to it, otherwise we go
# looking for it.
if(TARGET sodium)
    target_link_libraries(coinevomq PRIVATE sodium)
    if(coinevomq_build_static_libzmq)
        target_link_libraries(libzmq_vendor INTERFACE sodium)
    endif()
else()
    pkg_check_modules(sodium REQUIRED libsodium IMPORTED_TARGET)

    if(BUILD_SHARED_LIBS)
        target_link_libraries(coinevomq PRIVATE PkgConfig::sodium)
        if(coinevomq_build_static_libzmq)
            target_link_libraries(libzmq_vendor INTERFACE PkgConfig::sodium)
        endif()
    else()
        link_dep_libs(coinevomq PRIVATE "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
        target_include_directories(coinevomq PRIVATE ${sodium_STATIC_INCLUDE_DIRS})
        if(coinevomq_build_static_libzmq)
            link_dep_libs(libzmq_vendor INTERFACE "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
            target_link_libraries(libzmq_vendor INTERFACE ${sodium_STATIC_INCLUDE_DIRS})
        endif()
    endif()
endif()

add_library(coinevomq::coinevomq ALIAS coinevomq)

export(
    TARGETS coinevomq
    NAMESPACE coinevomq::
    FILE coinevomqTargets.cmake
)
install(
    TARGETS coinevomq
    EXPORT coinevomqConfig
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(
    FILES coinevomq/auth.h
          coinevomq/batch.h
          coinevomq/bt_serialize.h
          coinevomq/connections.h
          coinevomq/hex.h
          coinevomq/coinevomq.h
          coinevomq/message.h
          coinevomq/string_view.h
          ${CMAKE_CURRENT_BINARY_DIR}/coinevomq/version.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/coinevomq
)
option(COINEVOMQ_INSTALL_MAPBOX_VARIANT "Install mapbox-variant headers with coinevomq/ headers" ON)
if(COINEVOMQ_INSTALL_MAPBOX_VARIANT)
  install(
    FILES mapbox-variant/include/mapbox/variant.hpp
          mapbox-variant/include/mapbox/variant_cast.hpp
          mapbox-variant/include/mapbox/variant_io.hpp
          mapbox-variant/include/mapbox/variant_visitor.hpp
          mapbox-variant/include/mapbox/recursive_wrapper.hpp
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/coinevomq/mapbox
  )
endif()

option(COINEVOMQ_INSTALL_CPPZMQ "Install cppzmq header with coinevomq/ headers" ON)
if(COINEVOMQ_INSTALL_CPPZMQ)
  install(
    FILES cppzmq/zmq.hpp
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/coinevomq
  )
endif()


install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/libcoinevomq.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
  set(coinevomq_IS_TOPLEVEL_PROJECT TRUE)
else()
  set(coinevomq_IS_TOPLEVEL_PROJECT FALSE)
endif()

option(COINEVOMQ_BUILD_TESTS "Building and perform coinevomq tests" ${coinevomq_IS_TOPLEVEL_PROJECT})
if(COINEVOMQ_BUILD_TESTS)
    add_subdirectory(tests)
endif()

