cmake_minimum_required(VERSION 3.10)
project(vmorc CXX)

message( "NVM::VMORC v1")
message( "-----------------------------")
message(STATUS "NVM::VMORC LIB DIR:${CMAKE_CURRENT_SOURCE_DIR}")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})
set(ABSL_PROPAGATE_CXX_STD ON)

option(LIB_VMORC_BUILD_DOC "Build documentation" OFF)
option(LIB_VMORC_SANITIZE_ADDRESS "SANITIZE ADDRESS" ON)
option(LIB_VMORC_USE_CATCH ON)
option(LIB_VMORC_USE_LIB ON)
option(LIB_VMORC_USE_TEST OFF)

# Add ASAN
if(LIB_VMORC_SANITIZE_ADDRESS)
    set(SANITIZE_ADDRESS ON)
endif(LIB_VMORC_SANITIZE_ADDRESS)

#  make ourself sanitize and sane well.
if(NOT Sanitizers_FOUND)
 find_package(Sanitizers)
endif()

# Add Doxygen
if(LIB_VMORC_BUILD_DOC)
    if(NOT DOXYGEN_FOUND)
        find_package(Doxygen)
    endif()

    if (DOXYGEN_FOUND)
        # set input and output files
        set(DOXYGEN_IN ${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in)
        set(DOXYGEN_OUT ${CMAKE_CURRENT_SOURCE_DIR}/generated/docs-cached/Doxyfile)

        # request to configure the file
        configure_file(${DOXYGEN_IN} ${DOXYGEN_OUT} @ONLY)
        message("Doxygen build started")

         # note the option ALL which allows to build the docs together with the application
        add_custom_target( doc_vmorc ALL
            COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_OUT}
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/generated/docs-cached/
            COMMENT "Generating API documentation with Doxygen"
            VERBATIM )
    else (DOXYGEN_FOUND)
        message("Doxygen need to be installed to generate the doxygen documentation")
    endif (DOXYGEN_FOUND)
endif(LIB_VMORC_BUILD_DOC)

# Add Unit Test Lib Catch2
if(LIB_VMORC_USE_CATCH)
 message(STATUS  "CATCH2::ADD_LIB.")
 find_package(Catch2 3 REQUIRED)
endif()

# Add dependencies source code

# Google Abseil
add_subdirectory(deps/abseil-20230802.1 build-abseil)

# UTF8-CPP source code
file(GLOB_RECURSE SOURCES_UTF8CPP CONFIGURE_DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/deps/utfcpp-4.0.4/source/*.h
)

add_library(utf8cpp ${SOURCES_UTF8CPP})
set_target_properties(utf8cpp PROPERTIES LINKER_LANGUAGE CXX)
target_compile_features(utf8cpp PUBLIC cxx_std_17)
target_include_directories(utf8cpp
    PUBLIC
        deps/utfcpp-4.0.4/source/
)

# nvm nvm::core
add_subdirectory(deps/nvm-core build-nvm-core)

# Main headers and sources

# file(GLOB_RECURSE HEADERS_VMORC_MAIN CONFIGURE_DEPENDS
#     ${CMAKE_CURRENT_SOURCE_DIR}/vmorc/*.h
# )

add_library(vmorc_interface INTERFACE)
target_include_directories(vmorc_interface INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/vmorc
)


file(GLOB_RECURSE SOURCES_VMORC CONFIGURE_DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/vmorc/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/vmorc/*.s
)

add_library(${PROJECT_NAME}  ${SOURCES_VMORC} )
target_link_libraries(${PROJECT_NAME} 
    PUBLIC
    vmorc_interface  
    utf8cpp
    absl::base 
    absl::flat_hash_map 
    absl::algorithm
    absl::numeric
    absl::random_random
    absl::synchronization 
    absl::strings
    absl::time
    absl::utility)


target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_17)



add_executable(${PROJECT_NAME}-run main.cc)
if(Sanitizers_FOUND)
    add_sanitizers("${PROJECT_NAME}-run")
    message(STATUS "${PROJECT_NAME}-run_LIB::ASAN:ON.")
endif()

target_link_libraries(${PROJECT_NAME}-run PRIVATE  ${PROJECT_NAME} )


if(LIB_VMORC_USE_TEST)
    message(STATUS "TEST::ADD_UNIT_TEST.")
    add_subdirectory(tests)
endif()       

message(STATUS "Nativemq LIB Configuration Done!\n")
set(vmorc_FOUND  ON)