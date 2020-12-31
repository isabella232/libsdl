cmake_minimum_required(VERSION 3.5)
project(myapp)

set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/install)

message("PROJECT_NAME=" ${PROJECT_NAME})
message("CMAKE_INSTALL_PREFIX=" ${CMAKE_INSTALL_PREFIX})
message("CMAKE_SYSTEM_NAME=" ${CMAKE_SYSTEM_NAME})
message("CMAKE_BUILD_TYPE=" ${CMAKE_BUILD_TYPE})
message("CMAKE_CXX_STANDARD=" ${CMAKE_CXX_STANDARD})
message("CMAKE_CXX_FLAGS=" ${CMAKE_CXX_FLAGS})
message("CMAKE_C_FLAGS=" ${CMAKE_C_FLAGS})

if(WIN32)
message("WIN32=true")
endif()

if(MSVC)
message("MSVC=true")    
endif()

if(IOS)
message("IOS=true")    
endif()

if(APPLE)
message("APPLE=true")    
endif()

if(UNIX)
message("UNIX=true")    
endif()

add_subdirectory(src/libgui)
#add_subdirectory(src/test)

link_libraries(SDL2_test)
link_libraries(SDL2-static)
link_libraries(SDL2main)

# if(WINDOWS)
#     message("WINDOWS=true")
#     link_libraries(SDL2main)
#     add_definitions(-Dmain=main)
# endif()

add_executable(${PROJECT_NAME} src/main.cpp)

#set_target_properties(${PROJECT_NAME} PROPERTIES
#LINK_FLAGS /SUBSYSTEM:WINDOWS
#)

#set_target_properties(${PROJECT_NAME} PROPERTIES
#LINK_FLAGS /SUBSYSTEM:CONSOLE
#)

#target_include_directories(${PROJECT_NAME} PUBLIC libgui/include)
#target_link_libraries(${PROJECT_NAME} PUBLIC SDL2)
