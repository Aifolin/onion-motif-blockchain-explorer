cmake_minimum_required(VERSION 2.8)

set(PROJECT_NAME
        mtfblocks)


project(${PROJECT_NAME})

set(CMAKE_CXX_STANDARD 11)

if (WIN32)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wa,-mbig-obj -O3")
endif()


if (NOT MOTIF_DIR)
    set(MOTIF_DIR ~/motif)
endif()

message(STATUS MOTIF_DIR ": ${MOTIF_DIR}")

if (NOT MOTIF_SOURCE_DIR)
    set(MOTIF_SOURCE_DIR ${MOTIF_DIR}
          CACHE PATH "Path to the root directory for Motif")
endif()

if (NOT MOTIF_BUILD_DIR)
    # set location of motif build tree
    set(MOTIF_BUILD_DIR ${MOTIF_SOURCE_DIR}/build/Linux/master/release/
          CACHE PATH "Path to the build directory for Motif")
endif()

set(MY_CMAKE_DIR "${CMAKE_CURRENT_LIST_DIR}/cmake"
        CACHE PATH "The path to the cmake directory of the current project")

list(APPEND CMAKE_MODULE_PATH "${MY_CMAKE_DIR}")

set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${MOTIF_BUILD_DIR}"
        CACHE PATH "Add Motif directory for library searching")

include(MyUtils)

find_package(Motif)

# find boost
find_package(Boost COMPONENTS
        system
        filesystem
        thread
        date_time
        chrono
        regex
        serialization
        program_options
        date_time
        REQUIRED)

#info https://github.com/arsenm/sanitizers-cmake
find_package(Sanitizers)

if(APPLE)
    include_directories(/usr/local/opt/openssl/include)
    link_directories(/usr/local/opt/openssl/lib)
endif()


if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin" OR WIN32)
    add_library(unbound STATIC IMPORTED)
    set_property(TARGET unbound PROPERTY IMPORTED_LOCATION ${MOTIF_BUILD_DIR}/external/unbound/libunbound.a)
endif()



# include boost headers
include_directories(${Boost_INCLUDE_DIRS})

# include motif
include_directories(${MOTIF_SOURCE_DIR}/build)

include_directories("ext/mstch/include")
include_directories("ext/mstch/include/src")
include_directories("ext/crow")

# add ext/ subfolder
add_subdirectory(ext/)

# add src/ subfolder
add_subdirectory(src/)


set(SOURCE_FILES
        main.cpp)

#ADD_CUSTOM_TARGET(driver DEPENDS src/templates/index.html)

add_executable(${PROJECT_NAME}
        ${SOURCE_FILES})

add_sanitizers(${PROJECT_NAME})

create_git_version()

configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates ${CMAKE_CURRENT_BINARY_DIR}/templates)
configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates/css ${CMAKE_CURRENT_BINARY_DIR}/templates/css)
configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates/partials ${CMAKE_CURRENT_BINARY_DIR}/templates/partials)
configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates/js ${CMAKE_CURRENT_BINARY_DIR}/templates/js)

set(LIBRARIES
        myxrm
        myext
        mstch
        wallet
        blockchain_db
        cryptonote_core
        cryptonote_protocol
        cryptonote_basic
        multisig
        daemonizer
        cncrypto
        blocks
        lmdb
        ringct
        ringct_basic
        device
        common
        mnemonics
        easylogging
        checkpoints
        version
        epee
        sodium
        ${Boost_LIBRARIES}
        pthread
        unbound
        curl
        crypto
        ssl)

if(APPLE)
    set(LIBRARIES ${LIBRARIES} "-framework IOKit -framework Foundation")
else()
    set(LIBRARIES ${LIBRARIES} atomic)
endif()

if (NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin" AND NOT WIN32)
    set(LIBRARIES ${LIBRARIES} unwind)
endif()

if (WIN32)
    set(LIBRARIES ${LIBRARIES}
            wsock32
            ntdll
            ws2_32
            Iphlpapi
            )
else()
    set(LIBRARIES ${LIBRARIES} dl)
endif()

find_package(HIDAPI)
set(LIBRARIES ${LIBRARIES} ${HIDAPI_LIBRARIES})

target_link_libraries(${PROJECT_NAME} ${LIBRARIES})
