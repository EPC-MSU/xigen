CMAKE_MINIMUM_REQUIRED(VERSION 2.8.12)

PROJECT(xigen)

#set the right version here !

SET (XIGEN_VERSION_MAJOR 1)

SET (XIGEN_VERSION_MINOR 0)

SET (XIGEN_VERSION_PATCH 0) 

IF(${CMAKE_SYSTEM_NAME} STREQUAL Darwin)

    # It is required to build a rpath framework

    CMAKE_POLICY(SET CMP0042 NEW)

ENDIF()



# Detect correct C++11 flag

IF(NOT ${CMAKE_SYSTEM_NAME} STREQUAL Windows)

    INCLUDE(CheckCXXCompilerFlag)

    CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)

    CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)

    IF(COMPILER_SUPPORTS_CXX11)

        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")

    ELSEIF(COMPILER_SUPPORTS_CXX0X)

        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")

    ELSE()

        MESSAGE(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")

    ENDIF()

ENDIF()

#to build test and example in bin/folder and avoid ld errors

IF (NOT MSVC)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin)

ENDIF()  

IF(${CMAKE_SYSTEM_NAME} STREQUAL Windows AND ${CMAKE_SYSTEM_VERSION} EQUAL 6.1)

    # HACK: Make cryptopp replace missing (till Windows 8) GetOverlappedResultEx() on Windows7 with somehting compatible

    ADD_DEFINITIONS(-D_WIN32_WINNT=0x0601)

ENDIF()

IF(${CMAKE_SYSTEM_NAME} STREQUAL FreeBSD OR ${CMAKE_SYSTEM_NAME} STREQUAL Darwin)

    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")

ENDIF()

# set fPIC for static and shared libs

SET(CMAKE_POSITION_INDEPENDENT_CODE ON)

SET (SOURCES_STATIC
     src/driver.cc
     src/generator.cc
     src/locale.cc
     src/xigen.cc
    )

# Try to find BISON and FLEX

if (NOT MSVC)

    find_package(BISON)
    find_package(FLEX)

    BISON_TARGET(Parser ${CMAKE_SOURCE_DIR}/src/parser.yy ${CMAKE_CURRENT_BINARY_DIR}/parser.cc)

    FLEX_TARGET(Scanner ${CMAKE_SOURCE_DIR}/src/scanner.ll ${CMAKE_CURRENT_BINARY_DIR}/lexer.cc)

    ADD_FLEX_BISON_DEPENDENCY(Scanner Parser)

    add_executable(xigen
                   ${SOURCES_STATIC}
                   ${FLEX_Scanner_OUTPUTS}
                   ${BISON_Parser_OUTPUTS}
    )

    target_include_directories(xigen PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
    target_include_directories(xigen PRIVATE ${CMAKE_SOURCE_DIR}/src)
else()
    set_source_files_properties(${CMAKE_SOURCE_DIR}/src/parser.cc ${CMAKE_SOURCE_DIR}/src/scanner.cc
                                PROPERTIES GENERATED TRUE)
    add_executable(xigen
                   ${SOURCES_STATIC}
                   ${CMAKE_SOURCE_DIR}/src/compat/getopt_long.c
                   ${CMAKE_SOURCE_DIR}/src/parser.cc
                   ${CMAKE_SOURCE_DIR}/src/scanner.cc
                  )                            
    target_include_directories(xigen PRIVATE ${CMAKE_SOURCE_DIR}/src)
    target_include_directories(xigen PRIVATE ${CMAKE_SOURCE_DIR}/src/compat)
endif()

if(MSVC)

    target_compile_options(xigen PRIVATE /W3 /wd"4267" /wd"4244" /wd"4390" /wd"4800" /WX  /D"_CRT_SECURE_NO_WARNINGS" 
                           /D"_CRT_SECURE_NO_DEPRECATE" /D"_CRT_NONSTDC_NO_DEPRECATE" /D"_USING_V110_SDK71_" /D"_UNICODE" /D"UNICODE" /errorReport:prompt)
    add_custom_command(TARGET xigen PRE_BUILD
                       COMMAND c:/cygwin/bin/bash.exe --login -c "cd  `cygpath %27${CMAKE_SOURCE_DIR}/src%27` && flex -i -+ -oscanner.cc scanner.ll && bison -y -d -v -ra -S lalr1.cc -o parser.cc parser.yy"
    )
else()
    target_compile_options(xigen PRIVATE -Wno-deprecated-declarations -Wno-tautological-compare)
endif()
