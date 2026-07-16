cmake_minimum_required(VERSION 3.20)

project(
  geometry-central-surface-viewer
  VERSION 0.1.0
  DESCRIPTION "A small Geometry Central and Polyscope surface mesh viewer"
  LANGUAGES CXX)

find_package(Threads REQUIRED)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(GC_VIEWER_FETCH_DEPS
       "Fetch Geometry Central and Polyscope when installed packages are not available"
       ON)

find_package(GeometryCentral CONFIG QUIET)
find_package(polyscope CONFIG QUIET)

if((NOT TARGET geometry-central::geometry-central OR NOT TARGET polyscope)
   AND GC_VIEWER_FETCH_DEPS)
  include(FetchContent)

  set(BUILD_SHARED_LIBS
      OFF
      CACHE BOOL "" FORCE)

  FetchContent_Declare(
    geometry-central
    GIT_REPOSITORY https://github.com/nmwsharp/geometry-central.git
    GIT_TAG 019669ddabda05e0f71fa3587cfb3c1dadf19cb8)
  FetchContent_Declare(
    polyscope
    GIT_REPOSITORY https://github.com/nmwsharp/polyscope.git
    GIT_TAG 59da72df6517cab8379865899bdffdbc96171301)

  FetchContent_MakeAvailable(geometry-central polyscope)
endif()

if(TARGET geometry-central::geometry-central)
  set(GC_VIEWER_GEOMETRY_TARGET geometry-central::geometry-central)
elseif(TARGET geometry-central)
  set(GC_VIEWER_GEOMETRY_TARGET geometry-central)
else()
  message(
    FATAL_ERROR
      "Geometry Central target was not found. Install Geometry Central, or "
      "configure with -DGC_VIEWER_FETCH_DEPS=ON.")
endif()

if(NOT TARGET polyscope)
  message(
    FATAL_ERROR
      "Polyscope target was not found. Install Polyscope, or configure with "
      "-DGC_VIEWER_FETCH_DEPS=ON.")
endif()

set(RICCI_FLOW_BUILD_TESTS OFF)
set(RICCI_FLOW_BUILD_CLI ON)
add_subdirectory(ricci_flow)

add_executable(gc_surface_viewer src/main.cpp src/flatten.cpp)
target_link_libraries(gc_surface_viewer PRIVATE ${GC_VIEWER_GEOMETRY_TARGET}
                                                polyscope Threads::Threads)
add_dependencies(gc_surface_viewer ricci_flow_cli)
target_compile_definitions(
  gc_surface_viewer
  PRIVATE RICCI_FLOW_CLI_PATH="$<TARGET_FILE:ricci_flow_cli>")

if(MSVC)
  target_compile_options(gc_surface_viewer PRIVATE /W4)
else()
  target_compile_options(gc_surface_viewer PRIVATE -Wall -Wextra -Wpedantic)
endif()
