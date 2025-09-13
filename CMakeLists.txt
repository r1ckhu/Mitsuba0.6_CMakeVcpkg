cmake_minimum_required(VERSION 3.15) # 建议提高版本以获得更好的 vcpkg 和现代 CMake 支持

project(Mitsuba)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# --- 基本项目设置 ---
if(NOT COMPILE_THREAD)
    include(ProcessorCount)
    ProcessorCount(COMPILE_THREAD)
    message(STATUS "Using ${COMPILE_THREAD} threads for parallel builds.")
endif()

if(MSVC)
  add_compile_options(/MP) # 启用多进程编译
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(MITSUBA_ROOT_DIR ${CMAKE_CURRENT_SOURCE_DIR})
file(TO_CMAKE_PATH "${MITSUBA_ROOT_DIR}" MITSUBA_ROOT_DIR)
add_compile_definitions(MITSUBA_ROOT_DIR="${MITSUBA_ROOT_DIR}")

set(MITSUBA_BIN_DIR ${CMAKE_BINARY_DIR}/bin)
file(TO_CMAKE_PATH "${MITSUBA_BIN_DIR}" MITSUBA_BIN_DIR)
add_compile_definitions(MITSUBA_BIN_DIR="${MITSUBA_BIN_DIR}")


file(MAKE_DIRECTORY ${MITSUBA_BIN_DIR})

# 设置所有构建类型的输出目录
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${MITSUBA_BIN_DIR})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# --- 查找依赖 (由 vcpkg 提供) ---
# vcpkg 会自动处理以下 find_package 调用

find_package(Qt5 COMPONENTS Core Gui Widgets Xml Network OpenGL XmlPatterns REQUIRED)
find_package(ZLIB REQUIRED)
find_package(OpenEXR REQUIRED)
# Boost 版本由 vcpkg 控制。移除 CMake 中的硬编码版本以避免冲突。
find_package(Boost REQUIRED COMPONENTS filesystem thread chrono date_time atomic)
find_package(Imath REQUIRED) 
find_package(libjpeg-turbo REQUIRED)
find_package(XercesC REQUIRED)
find_package(PNG REQUIRED)
find_package(GLEW REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(FFTW3 CONFIG REQUIRED)
find_package(FFTW3f CONFIG REQUIRED)

# --- 查找 Python (系统安装) ---
find_package(Python COMPONENTS Interpreter Development REQUIRED)
set(PYTHON_VERSION_WITHOUT_DOT "python${Python_VERSION_MAJOR}${Python_VERSION_MINOR}")
message(STATUS "Found Python: ${Python_EXECUTABLE} (version ${Python_VERSION})")
# 查找与找到的 Python 版本匹配的 Boost::python 组件
find_package(Boost REQUIRED COMPONENTS ${PYTHON_VERSION_WITHOUT_DOT})


# --- 设置包含目录 ---
# find_package 已自动处理依赖的包含目录，只需添加项目本身的 include
include_directories("${MITSUBA_ROOT_DIR}/include")
# --- 为 OpenEXR/Imath 添加兼容性包含路径 ---
find_package(OpenEXR REQUIRED) # 确保 OpenEXR_INCLUDE_DIRS 变量可用
include_directories(${OpenEXR_INCLUDE_DIRS}) # 添加 .../include 自身
# 添加关键的子目录


find_package(Imath REQUIRED)
message(STATUS "Imath include directories: ${Imath_INCLUDE_DIRS}")
message(STATUS "Imath libraries: ${Imath_LIBRARIES}")
# include_directories(${Imath_INCLUDE_DIRS})

# --- 配置 Mitsuba 编译定义 ---
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

# --- 定义库列表 (使用现代 CMake 目标) ---
# 这些变量会在子目录的 CMakeLists.txt 中被使用
set(BASE_LIB_LIST ws2_32 Eigen3::Eigen)
set(QT_LIB_LIST Qt5::Core Qt5::Gui Qt5::Widgets Qt5::Xml Qt5::Network Qt5::OpenGL Qt5::XmlPatterns)
set(OPENEXR_LIB_LIST OpenEXR::OpenEXR)
set(BOOST_LIB_LIST Boost::${PYTHON_VERSION_WITHOUT_DOT} Boost::filesystem Boost::thread)
set(JEPG_LIB_LIST libjpeg-turbo::jpeg)
set(XERCES_LIB_LIST XercesC::XercesC)
set(PNG_LIB_LIST PNG::PNG) # 现代目标
set(FFTW_LIB_LIST FFTW3::fftw3f FFTW3::fftw3)
set(GL_LIB_LIST opengl32 glu32 GLEW::glew gdi32 user32) # 现代目标


# --- 添加子项目 ---
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

# 为插件目标设置特定的输出目录
# 注意：这会覆盖全局设置，所以在此之后添加的 target 会输出到这里
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

# 恢复默认的输出目录，以防后续有其他非插件目标
# set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${MITSUBA_BIN_DIR})

# --- 复制数据文件 (项目特定逻辑，予以保留) ---
file(MAKE_DIRECTORY ${MITSUBA_ROOT_DIR}/data)
file(COPY ${MITSUBA_ROOT_DIR}/data/schema DESTINATION ${MITSUBA_BIN_DIR}/data/)
file(COPY ${MITSUBA_ROOT_DIR}/data/ior DESTINATION ${MITSUBA_BIN_DIR}/data/)
file(COPY ${MITSUBA_ROOT_DIR}/data/microfacet DESTINATION ${MITSUBA_BIN_DIR}/data/)

# --- DLL 复制说明 ---
# vcpkg 会在构建时自动将依赖的 DLL 复制到输出目录（MITSUBA_BIN_DIR）。
# 手动复制 DLL 的代码不再需要，已移除。
# 对于 Qt 的平台插件 (platforms)，如果运行时出现问题，
# 推荐使用 Qt 官方的 windeployqt 工具进行部署，而不是手动复制。
# 例如: C:\Qt\5.15.2\msvc2019_64\bin\windeployqt.exe ${MITSUBA_BIN_DIR}/your_app.exe