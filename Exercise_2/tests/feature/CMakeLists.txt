# required minimum version of cmake
cmake_minimum_required(VERSION 3.2 FATAL_ERROR)

option(USE_STATIC_BOOST "Statically link Boost" ${WIN32})

# set the path to the additional cmake modules
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../../cmake/modules)

# set path to externals
set(CMAKE_INCLUDE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../../externals/include)
set(CMAKE_LIBRARY_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../../externals/lib)

# set report and documentation output paths
if(NOT REPORT_DIR)
  set(REPORT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../reports CACHE PATH "Reports directory")
endif(NOT REPORT_DIR)
if(NOT TESTS_REPORT_DIR)
  set(TESTS_REPORT_DIR ${REPORT_DIR}/tests CACHE PATH "Testresult output directory" )
endif(NOT TESTS_REPORT_DIR)

# create report and documentation output directories
file(MAKE_DIRECTORY
    ${REPORT_DIR}
    ${TESTS_REPORT_DIR}
)

set(DOC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../doc CACHE PATH "Documentation directory")

file(MAKE_DIRECTORY
    ${DOC_DIR}
)

if(NOT RUNTIME_OUTPUT_DIRECTORY)
  set(RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/../../bin)
endif()

set(TEST_RUNTIME_OUTPUT_DIRECTORY ${RUNTIME_OUTPUT_DIRECTORY}/tests)

file(MAKE_DIRECTORY
    ${RUNTIME_OUTPUT_DIRECTORY}
    ${TEST_RUNTIME_OUTPUT_DIRECTORY}
)

if(NOT TARGET internal::projectlib)
  include(${CMAKE_CURRENT_SOURCE_DIR}/../../project-export.cmake)
endif()

# Check for clang compiler
if(CMAKE_CXX_COMPILER MATCHES ".*clang")
  set(CMAKE_COMPILER_IS_CLANGCXX 1)
endif()

############################################
# Create a single feature test executable. #
# Without coverage measurement             #
############################################
#
# cucumber-cpp
#

find_package(Cuke REQUIRED)

function(register_feature_steps STEP_DEFINITION_FILE)
  get_filename_component(STEP_DEFINITION_FILE_NAME ${STEP_DEFINITION_FILE} NAME)
  message(STATUS "Adding " ${STEP_DEFINITION_FILE})
  # In order to generate step definitions from multiple source files
  # we need to use set CUKE_OBJECT_PREFIX variable for every
  # source file to compile
  # https://github.com/cucumber/cucumber-cpp/issues/36
  string(REPLACE "-" "" OBJ_PREFIX ${STEP_DEFINITION_FILE_NAME})
  set_source_files_properties(${STEP_DEFINITION_FILE}.cpp PROPERTIES COMPILE_FLAGS "-DCUKE_OBJECT_PREFIX=Obj${OBJ_PREFIX}")
  set(STEP_SOURCES ${STEP_SOURCES} ${STEP_DEFINITION_FILE}.cpp PARENT_SCOPE)
endfunction()

# TODO add your header files here
set(FEATURETEST_HEADER_FILES
    )

source_group("Header Files" FILES ${FEATURETEST_HEADER_FILES})

# TODO add all your step definition files here (source file without extension)
if(CUKE_FOUND)
  register_feature_steps(features/step_definitions/dummy_steps)
  register_feature_steps(features/step_definitions/mock_steps)
endif(CUKE_FOUND)

if(STEP_SOURCES)
  # Find all feature files
  FILE(GLOB_RECURSE FEATURE_FILES features *.feature)

  source_group("Feature Files" FILES ${FEATURE_FILES})

  add_executable(cucumber-cpp_wireserver
      ${STEP_SOURCES}
      ${FEATURETEST_HEADER_FILES}
      ${FEATURE_FILES}
  )
  add_executable(feature::wireserver ALIAS cucumber-cpp_wireserver)

  set_target_properties(cucumber-cpp_wireserver
      PROPERTIES
      RUNTIME_OUTPUT_DIRECTORY "${TEST_RUNTIME_OUTPUT_DIRECTORY}"
  )

  #
  # Boost
  #
  if(USE_STATIC_BOOST)
    set(Boost_USE_STATIC_LIBS ON)
  else(USE_STATIC_BOOST)
    target_compile_definitions(cucumber-cpp_wireserver
        PRIVATE
        BOOST_ALL_DYN_LINK
    )
  endif(USE_STATIC_BOOST)

  set(CORE_BOOST_LIBS thread system regex date_time program_options)

  find_package(Boost COMPONENTS  ${CORE_BOOST_LIBS} REQUIRED)

  target_include_directories(cucumber-cpp_wireserver
      SYSTEM
      PRIVATE
      ${Boost_INCLUDE_DIRS}
  )

  target_include_directories(cucumber-cpp_wireserver
      PRIVATE
      ${CMAKE_CURRENT_SOURCE_DIR}/features/step_definitions
      )

  target_link_libraries(cucumber-cpp_wireserver
      Cuke::Cucumber-cpp
      ${Boost_LIBRARIES}
      internal::projectlib
  )

  if(CMAKE_COMPILER_IS_CLANGCXX)
    target_compile_options(cucumber-cpp_wireserver
        PRIVATE
        -Wno-non-virtual-dtor
        -Wno-global-constructors
    )
  endif(CMAKE_COMPILER_IS_CLANGCXX)
  #
  # Google Test driver
  #
  find_package(GTest REQUIRED)

  if(GTEST_FOUND)
    target_include_directories(cucumber-cpp_wireserver
        SYSTEM
        PRIVATE
        ${GTEST_INCLUDE_DIRS}
    )
    target_link_libraries(cucumber-cpp_wireserver
        ${GTEST_LIBRARIES}
    )

    if(UNIX)
      find_package(Threads) # GTest needs this and it's a static library
      target_link_libraries(cucumber-cpp_wireserver
          ${CMAKE_THREAD_LIBS_INIT}
      )
    endif(UNIX)
  endif(GTEST_FOUND)

  #
  # For using Google Mock
  #
  find_package(GMock)

  if(GMOCK_FOUND)
    target_link_libraries(cucumber-cpp_wireserver
        GMock::GMock
        GMock::Main
    )
  endif(GMOCK_FOUND)

  if(WIN32)
    target_link_libraries(cucumber-cpp_wireserver
        ws2_32
    )
  endif(WIN32)


  ###############################
  # Installing gem dependencies
  ###############################
  find_program(BUNDLE bundle)
  if(BUNDLE)
    message(STATUS "Installing gem dependencies")
    execute_process(COMMAND ${BUNDLE} WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
  else(BUNDLE)
    message(WARNING "Could not find Bundler: skipping Ruby Gem management")
  endif(BUNDLE)

  ###############################
  # Create run targets if cucumber is installed
  ###############################
  find_program(CUCUMBER_RUBY cucumber)
  if(CUCUMBER_RUBY)
    message(STATUS "Found Cucumber")
    # Target: run feature tests
    # Run all features except features tagged  @wip or @proposal
    if(WIN32)
        add_custom_target(start_wireserver
          start /B $<TARGET_FILE:cucumber-cpp_wireserver>
          WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
          COMMENT "Start wireserver."
        )
    else(WIN32)
      add_custom_target(start_wireserver
        feature::wireserver &
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Start wireserver."
      )
    endif(WIN32)

    add_dependencies(start_wireserver
        feature::wireserver
    )

    add_custom_target(run_feature_test
        ${CUCUMBER_RUBY} -P --tags ~@wip --tags ~@proposal --no-color -f pretty -s features
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Running cucumber."
    )
    if(UNIX)
      add_dependencies(run_feature_test
          start_wireserver
      )
    endif(UNIX)

    # Target: run wip features
    # Run cucumber with --wip argument and run only features tagged @wip
    add_custom_target(run_wip_features
        ${CUCUMBER_RUBY} -P --tags @wip:3 --wip --tags ~@proposal --no-color -f pretty -s features
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Running cucumber (wip features)."
    )

    if(UNIX)
      add_dependencies(run_wip_features
          start_wireserver
      )
    endif(UNIX)

    # Target: document features
    # Run all features except features tagged @wip or @proposal and generate html output
    add_custom_target(document_features
        ${CUCUMBER_RUBY} -P --tags ~@wip --tags ~@proposal --no-color -f pretty --no-source -f html -o ${DOC_DIR}/Features.html features
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Running cucumber (features documentation)."
    )

    if(UNIX)
      add_dependencies(document_features
          start_wireserver
      )
    endif(UNIX)

  else(CUCUMBER_RUBY)
    message(WARNING "Could not find Cucumber: Install cucumber first in order to run tests")
  endif(CUCUMBER_RUBY)
endif(STEP_SOURCES)
