cmake_minimum_required(VERSION 3.16...3.26)

include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/common/bootstrap.cmake" NO_POLICY_SCOPE)

project(${_name} VERSION ${_version})

option(ENABLE_FRONTEND_API "Use obs-frontend-api for UI functionality" ON)
option(ENABLE_QT "Use Qt functionality" ON)

include(compilerconfig)
include(defaults)
include(helpers)

add_library(${CMAKE_PROJECT_NAME} MODULE)

find_package(libobs REQUIRED)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE OBS::libobs)

if(ENABLE_FRONTEND_API)
  find_package(obs-frontend-api REQUIRED)
  target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE OBS::obs-frontend-api)
endif()

if(ENABLE_QT)
  find_qt(COMPONENTS Widgets Core)
  target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE Qt::Core Qt::Widgets)
  target_compile_options(
    ${CMAKE_PROJECT_NAME} PRIVATE $<$<C_COMPILER_ID:Clang,AppleClang>:-Wno-quoted-include-in-framework-header
                                  -Wno-comma>)
  set_target_properties(
    ${CMAKE_PROJECT_NAME}
    PROPERTIES AUTOMOC ON
               AUTOUIC ON
               AUTORCC ON)
endif()

set(USE_SYSTEM_OPENCV
    OFF
    CACHE STRING "Use system OpenCV")

if(NOT OS_LINUX AND USE_SYSTEM_OPENCV)
  message(FATAL_ERROR "System OpenCV is only supported on Linux!")
endif()

if(USE_SYSTEM_OPENCV)
  find_package(OpenCV REQUIRED COMPONENTS core imgproc)
  target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE "${OpenCV_LIBRARIES}")
  target_include_directories(${CMAKE_PROJECT_NAME} SYSTEM PUBLIC "${OpenCV_INCLUDE_DIRS}")
else()
  include(cmake/FetchOpenCV.cmake)
  target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE OpenCV)
endif()

set(USE_SYSTEM_TESSERACT
    OFF
    CACHE STRING "Use system Tesseract")

if(NOT OS_LINUX AND USE_SYSTEM_TESSERACT)
  message(FATAL_ERROR "System Tesseract is only supported on Linux!")
endif()

if(USE_SYSTEM_TESSERACT)
  find_package(Tesseract)
  if(Tesseract_FOUND)
    target_link_directories(${CMAKE_PROJECT_NAME} PRIVATE "${Tesseract_LIBRARY_DIRS}")
    target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE "${Tesseract_LIBRARIES}")
    target_include_directories(${CMAKE_PROJECT_NAME} SYSTEM PUBLIC "${Tesseract_INCLUDE_DIRS}")
  else()
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(Tesseract REQUIRED IMPORTED_TARGET tesseract lept)
    target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE PkgConfig::Tesseract)
  endif()
else()
  include(cmake/FetchTesseract.cmake)
  target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE Tesseract)
endif()

include(cmake/BuildInja.cmake)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE inja)

target_sources(${CMAKE_PROJECT_NAME} PRIVATE src/plugin-main.c src/obs-utils.cpp src/tesseract-ocr-utils.cpp
                                             src/ocr-filter.cpp src/ocr-filter-info.c src/text-render-helper.cpp)

set_target_properties_plugin(${CMAKE_PROJECT_NAME} PROPERTIES OUTPUT_NAME ${_name})
