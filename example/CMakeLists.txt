cmake_minimum_required(VERSION 3.20)

project(example VERSION 0.1 LANGUAGES CXX)

#配置通用编译
set(CMAKE_CXX_STANDARD_REQUIRED ON)
if(APPLE)
    set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64" CACHE STRING "" FORCE)
endif()

#导入exmaple的QML位置，不然import example有时候会爆红
set(QML_IMPORT_PATH ${CMAKE_BINARY_DIR}/example CACHE STRING "Qt Creator extra QML import paths" FORCE)

#判断FluentUI库类型
if(FLUENTUI_BUILD_STATIC_LIB)
    add_definitions(-DFLUENTUI_BUILD_STATIC_LIB)
endif()

#设置可执行文件输出目录
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin/debug)
else()
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin/release)
endif()

#获取文件路径分隔符（解决执行命令的时候有些平台会报错）
file(TO_CMAKE_PATH "/" PATH_SEPARATOR)

#设置版本号
add_definitions(-DVERSION=1,4,0,0)

find_package(Qt6 REQUIRED COMPONENTS Quick Svg Network)

if(QT_VERSION VERSION_GREATER_EQUAL "6.3")
    qt_standard_project_setup()
else()
    set(CMAKE_AUTOMOC ON)
    set(CMAKE_AUTORCC ON)
    set(CMAKE_AUTOUIC ON)
endif()

#遍历所有Cpp文件
file(GLOB_RECURSE CPP_FILES *.cpp *.h)
foreach(filepath ${CPP_FILES})
    string(REPLACE "${CMAKE_CURRENT_SOURCE_DIR}/" "" filename ${filepath})
    list(APPEND sources_files ${filename})
endforeach(filepath)

#遍历所有qml文件
file(GLOB_RECURSE QML_PATHS *.qml)
foreach(filepath ${QML_PATHS})
    string(REPLACE "${CMAKE_CURRENT_SOURCE_DIR}/" "" filename ${filepath})
    list(APPEND qml_files ${filename})
endforeach(filepath)

#遍历所有资源文件
file(GLOB_RECURSE RES_PATHS *.png *.jpg *.svg *.ico *.ttf *.webp qmldir)
foreach(filepath ${RES_PATHS})
    string(REPLACE "${CMAKE_CURRENT_SOURCE_DIR}/" "" filename ${filepath})
    list(APPEND resource_files ${filename})
endforeach(filepath)

#添加可执行文件
if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
    qt_add_executable(example
        ${sources_files}
        example.rc
    )
else ()
    qt_add_executable(example
        ${sources_files}
    )
endif ()

#复制动态库到可执行文件同级目录下
if(WIN32)
    if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
        set(DLLPATH ${CMAKE_SOURCE_DIR}/3rdparty/msvc/*.dll)
    else()
        set(DLLPATH ${CMAKE_SOURCE_DIR}/3rdparty/mingw/*.dll)
    endif()
    string(REPLACE "/" ${PATH_SEPARATOR} DLLPATH "${DLLPATH}")
    file(GLOB DLL_FILES ${DLLPATH})
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy
        ${DLL_FILES}
        ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
    )
endif()

#添加qml模块
qt_add_qml_module(example
    URI "example"
    VERSION 1.0
    QML_FILES ${qml_files}
    RESOURCES ${resource_files}
)

#导入component头文件,不然通过QML_NAMED_ELEMENT生成的c++类会找不到头文件报错
target_include_directories(example PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src/component
)

#设置属性
set_target_properties(example PROPERTIES
    MACOSX_BUNDLE_GUI_IDENTIFIER my.example.com
    MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION}
    MACOSX_BUNDLE_SHORT_VERSION_STRING ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}
    MACOSX_BUNDLE TRUE
    WIN32_EXECUTABLE TRUE
)

#链接库
if (FLUENTUI_BUILD_STATIC_LIB)
    target_link_libraries(example PRIVATE
        Qt6::Quick
        Qt6::Svg
        Qt6::Network
        fluentui
        fluentuiplugin
        FramelessHelper::Core
        FramelessHelper::Quick
    )
else()
    target_link_libraries(example PRIVATE
        Qt6::Quick
        Qt6::Svg
        Qt6::Network
        fluentuiplugin
        FramelessHelper::Core
        FramelessHelper::Quick
    )
endif()

#安装
install(TARGETS example
    BUNDLE DESTINATION .
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
