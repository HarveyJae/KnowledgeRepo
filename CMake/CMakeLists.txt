# 标准CMAKE文件写法以及相关指令使用方法
cmake_minimum_required(VERSION 3.5.0)
project(SpaceShoot VERSION 0.1 LANGUAGES C CXX)

# 设置C++标准
set(CMAKE_CXX_STANDARD 11)
# 编译器必须支持c++11，否则会报错
set(CMAKE_CXX_STANDARD_REQUIRED True)

# 设置编译选项
if (MSVC)
    add_compile_options(/W4)
else()
    add_compile_options(-Wall -Wextra -Werror)
endif()

# 设置编译输出目录
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_SOURCE_DIR}/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_SOURCE_DIR}/bin)


# 查找并载入SDL库的cmake预设，找不到会报错
find_package(SDL2 REQUIRED)
# 如果当前库不支持camke预设，那么需要导入库的头文件路径和库链接路径
# include_directories()
# link_directories()

# 设置可执行文件和平台相关
set(TARGET ${PROJECT_NAME}_${CMAKE_SYSTEM_NAME})
# 添加可执行文件
add_executable(${TARGET} main.cpp)

# SDL2是包名，使用find_package的时候必须在库前面加上包名
target_link_libraries(${TARGET}
    SDL2::SDL2
    SDL2::SDL2main
)
