cmake_minimum_required(VERSION 3.15) # Required for reliable vcpkg and modern CMake target support.

project(Mitsuba)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# --- Project setup ---
if(NOT COMPILE_THREAD)
    include(ProcessorCount)
    ProcessorCount(COMPILE_THREAD)
    message(STATUS "Using ${COMPILE_THREAD} threads for parallel builds.")
endif()

if(MSVC)
  add_compile_options(/MP) # Enable parallel compilation in Visual Studio.
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(MITSUBA_ROOT_DIR ${CMAKE_CURRENT_SOURCE_DIR})
file(TO_CMAKE_PATH "${MITSUBA_ROOT_DIR}" MITSUBA_ROOT_DIR)
add_compile_definitions(MITSUBA_ROOT_DIR="${MITSUBA_ROOT_DIR}")

set(MITSUBA_BIN_DIR ${CMAKE_BINARY_DIR}/bin)
file(TO_CMAKE_PATH "${MITSUBA_BIN_DIR}" MITSUBA_BIN_DIR)
add_compile_definitions(MITSUBA_BIN_DIR="${MITSUBA_BIN_DIR}")


file(MAKE_DIRECTORY ${MITSUBA_BIN_DIR})

# Keep runtime outputs in a single bin directory so tools can find plugins and data files.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${MITSUBA_BIN_DIR})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# --- Dependencies supplied by vcpkg ---
# The vcpkg toolchain resolves these packages and exposes imported targets.

find_package(Qt5 COMPONENTS Core Gui Widgets Xml Network OpenGL XmlPatterns REQUIRED)
find_package(ZLIB REQUIRED)
find_package(OpenEXR REQUIRED)
# Let vcpkg select the Boost version to avoid conflicts with local CMake overrides.
find_package(Boost REQUIRED COMPONENTS filesystem thread chrono date_time atomic)
find_package(Imath REQUIRED)
find_package(libjpeg-turbo REQUIRED)
find_package(XercesC REQUIRED)
find_package(PNG REQUIRED)
find_package(GLEW REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(FFTW3 CONFIG REQUIRED)
find_package(FFTW3f CONFIG REQUIRED)

# --- Python integration ---
find_package(Python COMPONENTS Interpreter Development REQUIRED)
set(PYTHON_VERSION_WITHOUT_DOT "python${Python_VERSION_MAJOR}${Python_VERSION_MINOR}")
message(STATUS "Found Python: ${Python_EXECUTABLE} (version ${Python_VERSION})")
# Match Boost.Python to the Python version found above.
find_package(Boost REQUIRED COMPONENTS ${PYTHON_VERSION_WITHOUT_DOT})


# --- Include paths ---
# Imported targets provide dependency include paths; only the project headers are added globally.
include_directories("${MITSUBA_ROOT_DIR}/include")
# --- OpenEXR/Imath compatibility include paths ---
find_package(OpenEXR REQUIRED) # Ensures OpenEXR_INCLUDE_DIRS is available for legacy source includes.
include_directories(${OpenEXR_INCLUDE_DIRS})


find_package(Imath REQUIRED)
message(STATUS "Imath include directories: ${Imath_INCLUDE_DIRS}")
message(STATUS "Imath libraries: ${Imath_LIBRARIES}")
# include_directories(${Imath_INCLUDE_DIRS})

# --- Mitsuba compile definitions ---
set(Mitsuba_Float_Percision "SINGLE_PRECISION")
set(Mitsuba_Spectrum_Samples "SPECTRUM_SAMPLES=3")
set(Mitsuba_SSE "MTS_SSE")
set(Mitsuba_Has_Coherent_RT "MTS_HAS_COHERENT_RT")
set(Mitsuba_Debug "MTS_DEBUG")
set(Mitsuba_OpenEXR "OPENEXR_DLL")

add_compile_definitions(
    ${Mitsuba_Float_Percision}
    ${Mitsuba_Spectrum_Samples}
    ${Mitsuba_SSE}
    ${Mitsuba_Has_Coherent_RT}
    ${Mitsuba_Debug}
    ${Mitsuba_OpenEXR}
    MTS_HAS_LIBPNG=1
    MTS_HAS_LIBJPEG=1
    MTS_HAS_OPENEXR=1
    MTS_HAS_FFTW=1
    DEBUG
    _CONSOLE
)

if(MSVC)
    add_compile_options(/openmp)
    add_link_options(/FIXED:NO /SUBSYSTEM:CONSOLE /NODEFAULTLIB:LIBCMT /MANIFEST)
endif()

# --- Shared library lists ---
# Subdirectory CMake files consume these variables when linking Mitsuba components.
set(BASE_LIB_LIST ws2_32 Eigen3::Eigen)
set(QT_LIB_LIST Qt5::Core Qt5::Gui Qt5::Widgets Qt5::Xml Qt5::Network Qt5::OpenGL Qt5::XmlPatterns)
set(OPENEXR_LIB_LIST OpenEXR::OpenEXR)
set(BOOST_LIB_LIST Boost::${PYTHON_VERSION_WITHOUT_DOT} Boost::filesystem Boost::thread)
set(JEPG_LIB_LIST libjpeg-turbo::jpeg)
set(XERCES_LIB_LIST XercesC::XercesC)
set(PNG_LIB_LIST PNG::PNG)
set(FFTW_LIB_LIST FFTW3::fftw3f FFTW3::fftw3)
set(GL_LIB_LIST opengl32 glu32 GLEW::glew gdi32 user32)


# --- Core targets ---
add_subdirectory(src/libcore)
add_subdirectory(src/librender)
add_subdirectory(src/libhw)
add_subdirectory(src/libbidir)
add_subdirectory(src/libpython)
add_subdirectory(src/mitsuba)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)
add_subdirectory(src/mtsgui)
set(CMAKE_AUTOMOC OFF)
set(CMAKE_AUTORCC OFF)
set(CMAKE_AUTOUIC OFF)

set(MITSUBA_PLUGIN_BIN_DIR ${MITSUBA_BIN_DIR}/plugins)
file(MAKE_DIRECTORY ${MITSUBA_PLUGIN_BIN_DIR})

message(STATUS "MITSUBA_BIN_DIR = ${MITSUBA_BIN_DIR}")
message(STATUS "MITSUBA_PLUGIN_BIN_DIR = ${MITSUBA_PLUGIN_BIN_DIR}")

# Plugin targets are added after this point and should be emitted under bin/plugins.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${MITSUBA_PLUGIN_BIN_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${MITSUBA_PLUGIN_BIN_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${MITSUBA_PLUGIN_BIN_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${MITSUBA_PLUGIN_BIN_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${MITSUBA_PLUGIN_BIN_DIR})
add_subdirectory(src/utils)
add_subdirectory(src/bsdfs)
add_subdirectory(src/phase)
add_subdirectory(src/shapes)
add_subdirectory(src/samplers)
add_subdirectory(src/rfilters)
add_subdirectory(src/films)
add_subdirectory(src/sensors)
add_subdirectory(src/emitters)
add_subdirectory(src/medium)
add_subdirectory(src/volume)
add_subdirectory(src/subsurface)
add_subdirectory(src/textures)
add_subdirectory(src/integrators)
add_subdirectory(src/tests)

# Restore the default runtime directory here if more non-plugin targets are added later.
# set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${MITSUBA_BIN_DIR})

# --- Runtime data files ---
file(MAKE_DIRECTORY ${MITSUBA_ROOT_DIR}/data)
file(COPY ${MITSUBA_ROOT_DIR}/data/schema DESTINATION ${MITSUBA_BIN_DIR}/data/)
file(COPY ${MITSUBA_ROOT_DIR}/data/ior DESTINATION ${MITSUBA_BIN_DIR}/data/)
file(COPY ${MITSUBA_ROOT_DIR}/data/microfacet DESTINATION ${MITSUBA_BIN_DIR}/data/)

# --- Deployment note ---
# vcpkg copies dependent DLLs into MITSUBA_BIN_DIR during the build.
# If Qt platform plugins are missing at runtime, deploy the executable with Qt's windeployqt tool.
