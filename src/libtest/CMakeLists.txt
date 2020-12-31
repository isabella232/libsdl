if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "Prevented in-tree built. Please create a build directory outside of the SDL source code and call cmake from there")
endif()

cmake_minimum_required(VERSION 3.0.0)
project(SDL2 C CXX)

if(WINDOWS_STORE)
  enable_language(CXX)
  cmake_minimum_required(VERSION 3.11)
  add_definitions(-DSDL_BUILDING_WINRT=1 -ZW)
  link_libraries(
    -nodefaultlib:vccorlib$<$<CONFIG:Debug>:d>
    -nodefaultlib:msvcrt$<$<CONFIG:Debug>:d>
    vccorlib$<$<CONFIG:Debug>:d>.lib
    msvcrt$<$<CONFIG:Debug>:d>.lib
  )
endif()

# !!! FIXME: this should probably do "MACOSX_RPATH ON" as a target property
# !!! FIXME:  for the SDL2 shared library (so you get an
# !!! FIXME:  install_name ("soname") of "@rpath/libSDL-whatever.dylib"
# !!! FIXME:  instead of "/usr/local/lib/libSDL-whatever.dylib"), but I'm
# !!! FIXME:  punting for now and leaving the existing behavior. Until this
# !!! FIXME:  properly resolved, this line silences a warning in CMake 3.0+.
# !!! FIXME:  remove it and this comment entirely once the problem is
# !!! FIXME:  properly resolved.
#cmake_policy(SET CMP0042 OLD)

include(CheckLibraryExists)
include(CheckIncludeFiles)
include(CheckIncludeFile)
include(CheckSymbolExists)
include(CheckCSourceCompiles)
include(CheckCSourceRuns)
include(CheckCCompilerFlag)
include(CheckTypeSize)
include(CheckStructHasMember)
include(CMakeDependentOption)
include(FindPkgConfig)
include(GNUInstallDirs)
set(CMAKE_MODULE_PATH "${SDL2_SOURCE_DIR}/cmake")
include(${SDL2_SOURCE_DIR}/cmake/macros.cmake)
include(${SDL2_SOURCE_DIR}/cmake/sdlchecks.cmake)

# General settings
# Edit include/SDL_version.h and change the version, then:
#   SDL_MICRO_VERSION += 1;
#   SDL_INTERFACE_AGE += 1;
#   SDL_BINARY_AGE += 1;
# if any functions have been added, set SDL_INTERFACE_AGE to 0.
# if backwards compatibility has been broken,
# set SDL_BINARY_AGE and SDL_INTERFACE_AGE to 0.
set(SDL_MAJOR_VERSION 2)
set(SDL_MINOR_VERSION 0)
set(SDL_MICRO_VERSION 14)
set(SDL_INTERFACE_AGE 0)
set(SDL_BINARY_AGE 14)
set(SDL_VERSION "${SDL_MAJOR_VERSION}.${SDL_MINOR_VERSION}.${SDL_MICRO_VERSION}")
# the following should match the versions in Xcode project file:
set(DYLIB_CURRENT_VERSION 15.0.0)
set(DYLIB_COMPATIBILITY_VERSION 1.0.0)

# Set defaults preventing destination file conflicts
set(SDL_CMAKE_DEBUG_POSTFIX "d"
    CACHE STRING "Name suffix for debug builds")

mark_as_advanced(CMAKE_IMPORT_LIBRARY_SUFFIX SDL_CMAKE_DEBUG_POSTFIX)

# Calculate a libtool-like version number
math(EXPR LT_CURRENT "${SDL_MICRO_VERSION} - ${SDL_INTERFACE_AGE}")
math(EXPR LT_AGE "${SDL_BINARY_AGE} - ${SDL_INTERFACE_AGE}")
math(EXPR LT_MAJOR "${LT_CURRENT}- ${LT_AGE}")
set(LT_REVISION "${SDL_INTERFACE_AGE}")
set(LT_RELEASE "${SDL_MAJOR_VERSION}.${SDL_MINOR_VERSION}")
set(LT_VERSION "${LT_MAJOR}.${LT_AGE}.${LT_REVISION}")

#message(STATUS "${LT_VERSION} :: ${LT_AGE} :: ${LT_REVISION} :: ${LT_CURRENT} :: ${LT_RELEASE}")

# General settings & flags
set(LIBRARY_OUTPUT_DIRECTORY "build")
# Check for 64 or 32 bit
set(SIZEOF_VOIDP ${CMAKE_SIZEOF_VOID_P})
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(ARCH_64 TRUE)
  set(PROCESSOR_ARCH "x64")
else()
  set(ARCH_64 FALSE)
  set(PROCESSOR_ARCH "x86")
endif()
set(LIBNAME SDL2)
if(NOT LIBTYPE)
  set(LIBTYPE SHARED)
endif()

# Get the platform
if(WIN32)
  if(NOT WINDOWS)
    set(WINDOWS TRUE)
  endif()
elseif(UNIX AND NOT APPLE)
  if(CMAKE_SYSTEM_NAME MATCHES ".*Linux")
    set(LINUX TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES "kFreeBSD.*")
    set(FREEBSD TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES "kNetBSD.*|NetBSD.*")
    set(NETBSD TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES "kOpenBSD.*|OpenBSD.*")
    set(OPENBSD TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES ".*GNU.*")
    set(GNU TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES ".*BSDI.*")
    set(BSDI TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES "DragonFly.*|FreeBSD")
    set(FREEBSD TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES "SYSV5.*")
    set(SYSV5 TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES "Solaris.*")
    set(SOLARIS TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES "HP-UX.*")
    set(HPUX TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES "AIX.*")
    set(AIX TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES "Minix.*")
    set(MINIX TRUE)
  endif()
elseif(APPLE)
  if(CMAKE_SYSTEM_NAME MATCHES ".*Darwin.*")
    set(DARWIN TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES ".*MacOS.*")
    set(MACOSX TRUE)
  elseif(CMAKE_SYSTEM_NAME MATCHES ".*tvOS.*")
    set(TVOS TRUE)
  endif()
  # TODO: iOS?
elseif(CMAKE_SYSTEM_NAME MATCHES "BeOS.*")
  message_error("BeOS support has been removed as of SDL 2.0.2.")
elseif(CMAKE_SYSTEM_NAME MATCHES "Haiku.*")
  set(HAIKU TRUE)
endif()

# Don't mistake osx for unix
if(UNIX AND NOT APPLE AND NOT RISCOS)
  set(UNIX_SYS ON)
else()
  set(UNIX_SYS OFF)
endif()

if(UNIX OR APPLE)
  set(UNIX_OR_MAC_SYS ON)
else()
  set(UNIX_OR_MAC_SYS OFF)
endif()

if (UNIX_OR_MAC_SYS AND NOT EMSCRIPTEN) # JavaScript does not yet have threading support, so disable pthreads when building for Emscripten.
  set(SDL_PTHREADS_ENABLED_BY_DEFAULT ON)
else()
  set(SDL_PTHREADS_ENABLED_BY_DEFAULT OFF)
endif()

# Default option knobs
if(APPLE OR ARCH_64)
  if(NOT "${CMAKE_OSX_ARCHITECTURES}" MATCHES "arm")
    set(OPT_DEF_SSEMATH ON)
  endif()
endif()
if(UNIX OR MINGW OR MSYS)
  set(OPT_DEF_LIBC ON)
endif()

# The hidraw support doesn't catch Xbox, PS4 and Nintendo controllers,
#  so we'll just use libusb when it's available. libusb does not support iOS,
#  so we default to yes on iOS.
#  TODO: Windows can support libusb, the hid.c file just depends on Unix APIs
if((WINDOWS AND NOT WINDOWS_STORE) OR IOS OR TVOS OR ANDROID)
  set(HIDAPI_SKIP_LIBUSB TRUE)
else()
  set(HIDAPI_SKIP_LIBUSB FALSE)
endif()
if (HIDAPI_SKIP_LIBUSB)
  set(OPT_DEF_HIDAPI ON)
endif()

# On the other hand, *BSD specifically uses libusb only, so we make a special
#  case just for them.
if(FREEBSD OR NETBSD OR OPENBSD OR BSDI)
  set(HIDAPI_ONLY_LIBUSB TRUE)
else()
  set(HIDAPI_ONLY_LIBUSB FALSE)
endif()

# Compiler info
if(CMAKE_COMPILER_IS_GNUCC)
  set(USE_GCC TRUE)
  set(OPT_DEF_ASM TRUE)
elseif(CMAKE_C_COMPILER_ID MATCHES "Clang")
  set(USE_CLANG TRUE)
  set(OPT_DEF_ASM TRUE)
elseif(MSVC_VERSION GREATER 1400) # VisualStudio 8.0+
  set(OPT_DEF_ASM TRUE)
  #set(CMAKE_C_FLAGS "/ZI /WX- /
else()
  set(OPT_DEF_ASM FALSE)
endif()

if(USE_GCC OR USE_CLANG)
  set(OPT_DEF_GCC_ATOMICS ON)
endif()

# Default flags, if not set otherwise
if("$ENV{CFLAGS}" STREQUAL "")
  if(CMAKE_BUILD_TYPE STREQUAL "")
    if(USE_GCC OR USE_CLANG)
      set(CMAKE_C_FLAGS "-g -O3")
    endif()
  endif()
else()
  set(CMAKE_C_FLAGS "$ENV{CFLAGS}")
  list(APPEND EXTRA_CFLAGS "$ENV{CFLAGS}")
endif()
if(NOT ("$ENV{CFLAGS}" STREQUAL "")) # Hackish, but does the trick on Win32
  list(APPEND EXTRA_LDFLAGS "$ENV{LDFLAGS}")
endif()

if(MSVC)
  option(FORCE_STATIC_VCRT "Force /MT for static VC runtimes" OFF)
  if(FORCE_STATIC_VCRT)
    foreach(flag_var
        CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
        CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)
      if(${flag_var} MATCHES "/MD")
        string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
      endif()
    endforeach()
  endif()

  # Make sure /RTC1 is disabled, otherwise it will use functions from the CRT
  foreach(flag_var
      CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
      CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)
    string(REGEX REPLACE "/RTC(su|[1su])" "" ${flag_var} "${${flag_var}}")
  endforeach(flag_var)
endif()

# Those are used for pkg-config and friends, so that the SDL2.pc, sdl2-config,
# etc. are created correctly.
set(SDL_LIBS "-lSDL2")
set(SDL_CFLAGS "")

# When building shared lib for Windows with MinGW,
# avoid the DLL having a "lib" prefix
if(WINDOWS)
  set(CMAKE_SHARED_LIBRARY_PREFIX "")
endif()

# Emscripten toolchain has a nonempty default value for this, and the checks
# in this file need to change that, so remember the original value, and
# restore back to that afterwards. For check_function_exists() to work in
# Emscripten, this value must be at its default value.
set(ORIG_CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS})

if(CYGWIN)
  # We build SDL on cygwin without the UNIX emulation layer
  include_directories("-I/usr/include/mingw")
  set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -mno-cygwin")
  check_c_source_compiles("int main(int argc, char **argv) {}"
    HAVE_GCC_NO_CYGWIN)
  set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})
  if(HAVE_GCC_NO_CYGWIN)
    list(APPEND EXTRA_LDFLAGS "-mno-cygwin")
    list(APPEND SDL_LIBS "-mno-cygwin")
  endif()
  set(SDL_CFLAGS "${SDL_CFLAGS} -I/usr/include/mingw")
endif()

add_definitions(-DUSING_GENERATED_CONFIG_H)
# General includes
include_directories(${SDL2_BINARY_DIR}/include ${SDL2_SOURCE_DIR}/include)
if(USE_GCC OR USE_CLANG)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -idirafter \"${SDL2_SOURCE_DIR}/src/video/khronos\"")
else()
  include_directories(${SDL2_SOURCE_DIR}/src/video/khronos)
endif()

# All these ENABLED_BY_DEFAULT vars will default to ON if not specified, so
#  you only need to have a platform override them if they are disabling.
set(OPT_DEF_ASM TRUE)
if(EMSCRIPTEN)
  # Set up default values for the currently supported set of subsystems:
  # Emscripten/Javascript does not have assembly support, a dynamic library
  # loading architecture, low-level CPU inspection or multithreading.
  set(OPT_DEF_ASM FALSE)
  set(SDL_SHARED_ENABLED_BY_DEFAULT OFF)
  set(SDL_ATOMIC_ENABLED_BY_DEFAULT OFF)
  set(SDL_THREADS_ENABLED_BY_DEFAULT OFF)
  set(SDL_LOADSO_ENABLED_BY_DEFAULT OFF)
  set(SDL_CPUINFO_ENABLED_BY_DEFAULT OFF)
  set(SDL_DLOPEN_ENABLED_BY_DEFAULT OFF)
endif()

# When defined, respect CMake's BUILD_SHARED_LIBS setting:
set(SDL_STATIC_ENABLED_BY_DEFAULT OFF)
if (NOT DEFINED SDL_SHARED_ENABLED_BY_DEFAULT)
  # ...unless decided already (as for EMSCRIPTEN)

  set(SDL_SHARED_ENABLED_BY_DEFAULT OFF)

  if (NOT DEFINED BUILD_SHARED_LIBS)
    # No preference? Build both, just like the AC/AM configure
    set(SDL_SHARED_ENABLED_BY_DEFAULT ON)

  elseif (BUILD_SHARED_LIBS)
    # In this case, we assume the user wants a shared lib and don't build
    # the static one
    set(SDL_SHARED_ENABLED_BY_DEFAULT ON)
    set(SDL_STATIC_ENABLED_BY_DEFAULT ON)
  endif()
endif()

set(SDL_SHARED_ENABLED_BY_DEFAULT OFF)
set(SDL_STATIC_ENABLED_BY_DEFAULT ON)

set(SDL_SUBSYSTEMS
    Atomic Audio Video Render Events Joystick Haptic Power Threads Timers
    File Loadso CPUinfo Filesystem Dlopen Sensor Locale)
foreach(_SUB ${SDL_SUBSYSTEMS})
  string(TOUPPER ${_SUB} _OPT)
  if (NOT DEFINED SDL_${_OPT}_ENABLED_BY_DEFAULT)
    set(SDL_${_OPT}_ENABLED_BY_DEFAULT ON)
  endif()
  option(SDL_${_OPT} "Enable the ${_SUB} subsystem" ${SDL_${_OPT}_ENABLED_BY_DEFAULT})
endforeach()

option_string(ASSERTIONS "Enable internal sanity checks (auto/disabled/release/enabled/paranoid)" "auto")
#set_option(DEPENDENCY_TRACKING "Use gcc -MMD -MT dependency tracking" ON)
set_option(LIBC                "Use the system C library" ${OPT_DEF_LIBC})
set_option(GCC_ATOMICS         "Use gcc builtin atomics" ${OPT_DEF_GCC_ATOMICS})
set_option(ASSEMBLY            "Enable assembly routines" ${OPT_DEF_ASM})
set_option(SSEMATH             "Allow GCC to use SSE floating point math" ${OPT_DEF_SSEMATH})
set_option(MMX                 "Use MMX assembly routines" ${OPT_DEF_ASM})
set_option(3DNOW               "Use 3Dnow! MMX assembly routines" ${OPT_DEF_ASM})
set_option(SSE                 "Use SSE assembly routines" ${OPT_DEF_ASM})
set_option(SSE2                "Use SSE2 assembly routines" ${OPT_DEF_SSEMATH})
set_option(SSE3                "Use SSE3 assembly routines" ${OPT_DEF_SSEMATH})
set_option(ALTIVEC             "Use Altivec assembly routines" ${OPT_DEF_ASM})
set_option(ARMSIMD             "use SIMD assembly blitters on ARM" OFF)
set_option(ARMNEON             "use NEON assembly blitters on ARM" OFF)
set_option(DISKAUDIO           "Support the disk writer audio driver" ON)
set_option(DUMMYAUDIO          "Support the dummy audio driver" ON)
set_option(VIDEO_DIRECTFB      "Use DirectFB video driver" OFF)
dep_option(DIRECTFB_SHARED     "Dynamically load directfb support" ON "VIDEO_DIRECTFB" OFF)
set_option(VIDEO_DUMMY         "Use dummy video driver" ON)
set_option(VIDEO_OPENGL        "Include OpenGL support" ON)
set_option(VIDEO_OPENGLES      "Include OpenGL ES support" ON)
set_option(PTHREADS            "Use POSIX threads for multi-threading" ${SDL_PTHREADS_ENABLED_BY_DEFAULT})
dep_option(PTHREADS_SEM        "Use pthread semaphores" ON "PTHREADS" OFF)
set_option(SDL_DLOPEN          "Use dlopen for shared object loading" ${SDL_DLOPEN_ENABLED_BY_DEFAULT})
dep_option(OSS                 "Support the OSS audio API" ON "UNIX_SYS OR RISCOS" OFF)
set_option(ALSA                "Support the ALSA audio API" ${UNIX_SYS})
dep_option(ALSA_SHARED         "Dynamically load ALSA audio support" ON "ALSA" OFF)
set_option(JACK                "Support the JACK audio API" ${UNIX_SYS})
dep_option(JACK_SHARED         "Dynamically load JACK audio support" ON "JACK" OFF)
set_option(ESD                 "Support the Enlightened Sound Daemon" ${UNIX_SYS})
dep_option(ESD_SHARED          "Dynamically load ESD audio support" ON "ESD" OFF)
set_option(PULSEAUDIO          "Use PulseAudio" ${UNIX_SYS})
dep_option(PULSEAUDIO_SHARED   "Dynamically load PulseAudio support" ON "PULSEAUDIO" OFF)
set_option(ARTS                "Support the Analog Real Time Synthesizer" ${UNIX_SYS})
dep_option(ARTS_SHARED         "Dynamically load aRts audio support" ON "ARTS" OFF)
set_option(NAS                 "Support the NAS audio API" ${UNIX_SYS})
set_option(NAS_SHARED          "Dynamically load NAS audio API" ${UNIX_SYS})
set_option(SNDIO               "Support the sndio audio API" ${UNIX_SYS})
dep_option(SNDIO_SHARED        "Dynamically load the sndio audio API" ${UNIX_SYS} ON "SNDIO" OFF)
set_option(FUSIONSOUND         "Use FusionSound audio driver" OFF)
dep_option(FUSIONSOUND_SHARED  "Dynamically load fusionsound audio support" ON "FUSIONSOUND" OFF)
set_option(LIBSAMPLERATE       "Use libsamplerate for audio rate conversion" ${UNIX_SYS})
dep_option(LIBSAMPLERATE_SHARED "Dynamically load libsamplerate" ON "LIBSAMPLERATE" OFF)
set_option(RPATH               "Use an rpath when linking SDL" ${UNIX_SYS})
set_option(CLOCK_GETTIME       "Use clock_gettime() instead of gettimeofday()" OFF)
set_option(VIDEO_X11           "Use X11 video driver" ${UNIX_SYS})
set_option(VIDEO_WAYLAND       "Use Wayland video driver" ${UNIX_SYS})
dep_option(WAYLAND_SHARED      "Dynamically load Wayland support" ON "VIDEO_WAYLAND" OFF)
dep_option(VIDEO_WAYLAND_QT_TOUCH  "QtWayland server support for Wayland video driver" ON "VIDEO_WAYLAND" OFF)
set_option(VIDEO_RPI           "Use Raspberry Pi video driver" ${UNIX_SYS})
dep_option(X11_SHARED          "Dynamically load X11 support" ON "VIDEO_X11" OFF)
set(SDL_X11_OPTIONS Xcursor Xinerama XInput Xrandr Xscrnsaver XShape Xvm)
foreach(_SUB ${SDL_X11_OPTIONS})
  string(TOUPPER "VIDEO_X11_${_SUB}" _OPT)
  dep_option(${_OPT}           "Enable ${_SUB} support" ON "VIDEO_X11" OFF)
endforeach()
set_option(VIDEO_COCOA         "Use Cocoa video driver" ${APPLE})
set_option(DIRECTX             "Use DirectX for Windows audio/video" ${WINDOWS})
set_option(WASAPI              "Use the Windows WASAPI audio driver" ${WINDOWS})
set_option(RENDER_D3D          "Enable the Direct3D render driver" ${WINDOWS})
set_option(RENDER_METAL        "Enable the Metal render driver" ${APPLE})
set_option(VIDEO_VIVANTE       "Use Vivante EGL video driver" ${UNIX_SYS})
dep_option(VIDEO_VULKAN        "Enable Vulkan support" ON "ANDROID OR APPLE OR LINUX OR WINDOWS" OFF)
set_option(VIDEO_METAL         "Enable Metal support" ${APPLE})
set_option(VIDEO_KMSDRM        "Use KMS DRM video driver" ${UNIX_SYS})
dep_option(KMSDRM_SHARED       "Dynamically load KMS DRM support" ON "VIDEO_KMSDRM" OFF)
set_option(VIDEO_OFFSCREEN     "Use offscreen video driver" OFF)
option_string(BACKGROUNDING_SIGNAL "number to use for magic backgrounding signal or 'OFF'" "OFF")
option_string(FOREGROUNDING_SIGNAL "number to use for magic foregrounding signal or 'OFF'" "OFF")
set_option(HIDAPI              "Use HIDAPI for low level joystick drivers" ${OPT_DEF_HIDAPI})
set_option(JOYSTICK_VIRTUAL    "Enable the virtual-joystick driver" ON)

set(SDL_SHARED ${SDL_SHARED_ENABLED_BY_DEFAULT} CACHE BOOL "Build a shared version of the library")
set(SDL_STATIC ${SDL_STATIC_ENABLED_BY_DEFAULT} CACHE BOOL "Build a static version of the library")

dep_option(SDL_STATIC_PIC      "Static version of the library should be built with Position Independent Code" OFF "SDL_STATIC" OFF)
#set_option(SDL_TEST            "Build the test directory" ON)

# General source files
file(GLOB SOURCE_FILES
  ${SDL2_SOURCE_DIR}/src/*.c
  ${SDL2_SOURCE_DIR}/src/atomic/*.c
  ${SDL2_SOURCE_DIR}/src/audio/*.c
  ${SDL2_SOURCE_DIR}/src/cpuinfo/*.c
  ${SDL2_SOURCE_DIR}/src/dynapi/*.c
  ${SDL2_SOURCE_DIR}/src/events/*.c
  ${SDL2_SOURCE_DIR}/src/file/*.c
  ${SDL2_SOURCE_DIR}/src/haptic/*.c
  ${SDL2_SOURCE_DIR}/src/libm/*.c
  ${SDL2_SOURCE_DIR}/src/locale/*.c
  ${SDL2_SOURCE_DIR}/src/misc/*.c
  ${SDL2_SOURCE_DIR}/src/power/*.c
  ${SDL2_SOURCE_DIR}/src/render/*.c
  ${SDL2_SOURCE_DIR}/src/render/*/*.c
  ${SDL2_SOURCE_DIR}/src/sensor/*.c
  ${SDL2_SOURCE_DIR}/src/stdlib/*.c
  ${SDL2_SOURCE_DIR}/src/thread/*.c
  ${SDL2_SOURCE_DIR}/src/timer/*.c
  ${SDL2_SOURCE_DIR}/src/video/*.c
  ${SDL2_SOURCE_DIR}/src/video/yuv2rgb/*.c)


if(ASSERTIONS STREQUAL "auto")
  # Do nada - use optimization settings to determine the assertion level
elseif(ASSERTIONS STREQUAL "disabled")
  set(SDL_DEFAULT_ASSERT_LEVEL 0)
elseif(ASSERTIONS STREQUAL "release")
  set(SDL_DEFAULT_ASSERT_LEVEL 1)
elseif(ASSERTIONS STREQUAL "enabled")
  set(SDL_DEFAULT_ASSERT_LEVEL 2)
elseif(ASSERTIONS STREQUAL "paranoid")
  set(SDL_DEFAULT_ASSERT_LEVEL 3)
else()
  message_error("unknown assertion level")
endif()
set(HAVE_ASSERTIONS ${ASSERTIONS})

if(NOT BACKGROUNDING_SIGNAL STREQUAL "OFF")
  add_definitions("-DSDL_BACKGROUNDING_SIGNAL=${BACKGROUNDING_SIGNAL}")
endif()

if(NOT FOREGROUNDING_SIGNAL STREQUAL "OFF")
  add_definitions("-DSDL_FOREGROUNDING_SIGNAL=${FOREGROUNDING_SIGNAL}")
endif()

# Compiler option evaluation
if(USE_GCC OR USE_CLANG)
  # Check for -Wall first, so later things can override pieces of it.
  check_c_compiler_flag(-Wall HAVE_GCC_WALL)
  if(HAVE_GCC_WALL)
    list(APPEND EXTRA_CFLAGS "-Wall")
    if(HAIKU)
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-multichar")
    endif()
  endif()

  check_c_compiler_flag(-fno-strict-aliasing HAVE_GCC_NO_STRICT_ALIASING)
  if(HAVE_GCC_NO_STRICT_ALIASING)
    list(APPEND EXTRA_CFLAGS "-fno-strict-aliasing")
  endif()

  check_c_compiler_flag(-Wdeclaration-after-statement HAVE_GCC_WDECLARATION_AFTER_STATEMENT)
  if(HAVE_GCC_WDECLARATION_AFTER_STATEMENT)
    check_c_compiler_flag(-Werror=declaration-after-statement HAVE_GCC_WERROR_DECLARATION_AFTER_STATEMENT)
    if(HAVE_GCC_WERROR_DECLARATION_AFTER_STATEMENT)
      list(APPEND EXTRA_CFLAGS "-Werror=declaration-after-statement")
    endif()
    list(APPEND EXTRA_CFLAGS "-Wdeclaration-after-statement")
  endif()

  if(DEPENDENCY_TRACKING)
    check_c_source_compiles("
        #if !defined(__GNUC__) || __GNUC__ < 3
        #error Dependency tracking requires GCC 3.0 or newer
        #endif
        int main(int argc, char **argv) { }" HAVE_DEPENDENCY_TRACKING)
  endif()

  if(GCC_ATOMICS)
    check_c_source_compiles("int main(int argc, char **argv) {
        int a;
        void *x, *y, *z;
        __sync_lock_test_and_set(&a, 4);
        __sync_lock_test_and_set(&x, y);
        __sync_fetch_and_add(&a, 1);
        __sync_bool_compare_and_swap(&a, 5, 10);
        __sync_bool_compare_and_swap(&x, y, z); }" HAVE_GCC_ATOMICS)
    if(NOT HAVE_GCC_ATOMICS)
      check_c_source_compiles("int main(int argc, char **argv) {
          int a;
          __sync_lock_test_and_set(&a, 1);
          __sync_lock_release(&a); }" HAVE_GCC_SYNC_LOCK_TEST_AND_SET)
    endif()
  endif()

  set(CMAKE_REQUIRED_FLAGS "-mpreferred-stack-boundary=2")
  check_c_source_compiles("int x = 0; int main(int argc, char **argv) {}"
    HAVE_GCC_PREFERRED_STACK_BOUNDARY)
  set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})

  set(CMAKE_REQUIRED_FLAGS "-fvisibility=hidden -Werror")
  check_c_source_compiles("
      #if !defined(__GNUC__) || __GNUC__ < 4
      #error SDL only uses visibility attributes in GCC 4 or newer
      #endif
      int main(int argc, char **argv) {}" HAVE_GCC_FVISIBILITY)
  if(HAVE_GCC_FVISIBILITY)
    list(APPEND EXTRA_CFLAGS "-fvisibility=hidden")
  endif()
  set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})

  check_c_compiler_flag(-Wshadow HAVE_GCC_WSHADOW)
  if(HAVE_GCC_WSHADOW)
    list(APPEND EXTRA_CFLAGS "-Wshadow")
  endif()

  if(APPLE)
    list(APPEND EXTRA_LDFLAGS "-Wl,-undefined,error")
    list(APPEND EXTRA_LDFLAGS "-Wl,-compatibility_version,${DYLIB_COMPATIBILITY_VERSION}")
    list(APPEND EXTRA_LDFLAGS "-Wl,-current_version,${DYLIB_CURRENT_VERSION}")
  else()
    set(CMAKE_REQUIRED_FLAGS "-Wl,--no-undefined")
    check_c_compiler_flag("" HAVE_NO_UNDEFINED)
    set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})
    if(HAVE_NO_UNDEFINED)
      list(APPEND EXTRA_LDFLAGS "-Wl,--no-undefined")
    endif()
  endif()
endif()

if(ASSEMBLY)
  if(USE_GCC OR USE_CLANG)
    set(SDL_ASSEMBLY_ROUTINES 1)
    # TODO: Those all seem to be quite GCC specific - needs to be
    # reworked for better compiler support
    set(HAVE_ASSEMBLY TRUE)
    if(MMX)
      set(CMAKE_REQUIRED_FLAGS "-mmmx")
      check_c_source_compiles("
          #ifdef __MINGW32__
          #include <_mingw.h>
          #ifdef __MINGW64_VERSION_MAJOR
          #include <intrin.h>
          #else
          #include <mmintrin.h>
          #endif
          #else
          #include <mmintrin.h>
          #endif
          #ifndef __MMX__
          #error Assembler CPP flag not enabled
          #endif
          int main(int argc, char **argv) { }" HAVE_MMX)
      if(HAVE_MMX)
        list(APPEND EXTRA_CFLAGS "-mmmx")
      endif()
      set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})
    endif()

    if(3DNOW)
      set(CMAKE_REQUIRED_FLAGS "-m3dnow")
      check_c_source_compiles("
          #include <mm3dnow.h>
          #ifndef __3dNOW__
          #error Assembler CPP flag not enabled
          #endif
          int main(int argc, char **argv) {
            void *p = 0;
            _m_prefetch(p);
          }" HAVE_3DNOW)
      if(HAVE_3DNOW)
        list(APPEND EXTRA_CFLAGS "-m3dnow")
      endif()
      set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})
    endif()

    if(SSE)
      set(CMAKE_REQUIRED_FLAGS "-msse")
      check_c_source_compiles("
          #ifdef __MINGW32__
          #include <_mingw.h>
          #ifdef __MINGW64_VERSION_MAJOR
          #include <intrin.h>
          #else
          #include <xmmintrin.h>
          #endif
          #else
          #include <xmmintrin.h>
          #endif
          #ifndef __SSE__
          #error Assembler CPP flag not enabled
          #endif
          int main(int argc, char **argv) { }" HAVE_SSE)
      if(HAVE_SSE)
        list(APPEND EXTRA_CFLAGS "-msse")
      endif()
      set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})
    endif()

    if(SSE2)
      set(CMAKE_REQUIRED_FLAGS "-msse2")
      check_c_source_compiles("
          #ifdef __MINGW32__
          #include <_mingw.h>
          #ifdef __MINGW64_VERSION_MAJOR
          #include <intrin.h>
          #else
          #include <emmintrin.h>
          #endif
          #else
          #include <emmintrin.h>
          #endif
          #ifndef __SSE2__
          #error Assembler CPP flag not enabled
          #endif
          int main(int argc, char **argv) { }" HAVE_SSE2)
      if(HAVE_SSE2)
        list(APPEND EXTRA_CFLAGS "-msse2")
      endif()
      set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})
    endif()

    if(SSE3)
      set(CMAKE_REQUIRED_FLAGS "-msse3")
      check_c_source_compiles("
          #ifdef __MINGW32__
          #include <_mingw.h>
          #ifdef __MINGW64_VERSION_MAJOR
          #include <intrin.h>
          #else
          #include <pmmintrin.h>
          #endif
          #else
          #include <pmmintrin.h>
          #endif
          #ifndef __SSE3__
          #error Assembler CPP flag not enabled
          #endif
          int main(int argc, char **argv) { }" HAVE_SSE3)
      if(HAVE_SSE3)
        list(APPEND EXTRA_CFLAGS "-msse3")
      endif()
      set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})
    endif()

    if(NOT SSEMATH)
      if(SSE OR SSE2 OR SSE3)
        if(USE_GCC)
          check_c_compiler_flag(-mfpmath=387 HAVE_FP_387)
          if(HAVE_FP_387)
            list(APPEND EXTRA_CFLAGS "-mfpmath=387")
          endif()
        endif()
        set(HAVE_SSEMATH TRUE)
      endif()
    endif()

    check_include_file("immintrin.h" HAVE_IMMINTRIN_H)

    if(ALTIVEC)
      set(CMAKE_REQUIRED_FLAGS "-maltivec")
      check_c_source_compiles("
          #include <altivec.h>
          vector unsigned int vzero() {
              return vec_splat_u32(0);
          }
          int main(int argc, char **argv) { }" HAVE_ALTIVEC_H_HDR)
      check_c_source_compiles("
          vector unsigned int vzero() {
              return vec_splat_u32(0);
          }
          int main(int argc, char **argv) { }" HAVE_ALTIVEC)
      set(CMAKE_REQUIRED_FLAGS ${ORIG_CMAKE_REQUIRED_FLAGS})
      if(HAVE_ALTIVEC OR HAVE_ALTIVEC_H_HDR)
        set(HAVE_ALTIVEC TRUE) # if only HAVE_ALTIVEC_H_HDR is set
        list(APPEND EXTRA_CFLAGS "-maltivec")
        set(SDL_ALTIVEC_BLITTERS 1)
        if(HAVE_ALTIVEC_H_HDR)
          set(HAVE_ALTIVEC_H 1)
        endif()
      endif()
    endif()

    if(ARMSIMD)
      set(ORIG_CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS}")
      set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -x assembler-with-cpp")
      check_c_source_compiles("
        .text
        .arch armv6
        .object_arch armv4
        .arm
        .altmacro
        #ifndef __ARM_EABI__
        #error EABI is required (to be sure that calling conventions are compatible)
        #endif
        pld [r0]
        uqadd8 r0, r0, r0
      " ARMSIMD_FOUND)
      set(CMAKE_REQUIRED_FLAGS "${ORIG_CMAKE_REQUIRED_FLAGS}")

      if(ARMSIMD_FOUND)
        set(HAVE_ARMSIMD TRUE)
        set(SDL_ARM_SIMD_BLITTERS 1)
        file(GLOB ARMSIMD_SOURCES ${SDL2_SOURCE_DIR}/src/video/arm/pixman-arm-simd*.S)
        set(SOURCE_FILES ${SOURCE_FILES} ${ARMSIMD_SOURCES})
        set(WARN_ABOUT_ARM_SIMD_ASM_MIT TRUE)
      endif()
    endif()

    if(ARMNEON)
      set(ORIG_CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS}")
      set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -x assembler-with-cpp")
      check_c_source_compiles("
        .text
        .fpu neon
        .arch armv7a
        .object_arch armv4
        .eabi_attribute 10, 0
        .arm
        .altmacro
        #ifndef __ARM_EABI__
        #error EABI is required (to be sure that calling conventions are compatible)
        #endif
        pld [r0]
        vmovn.u16 d0, q0
      " ARMNEON_FOUND)
      set(CMAKE_REQUIRED_FLAGS "${ORIG_CMAKE_REQUIRED_FLAGS}")

      if(ARMNEON_FOUND)
        set(HAVE_ARMNEON TRUE)
        set(SDL_ARM_NEON_BLITTERS 1)
        file(GLOB ARMNEON_SOURCES ${SDL2_SOURCE_DIR}/src/video/arm/pixman-arm-neon*.S)
        set(SOURCE_FILES ${SOURCE_FILES} ${ARMNEON_SOURCES})
        set(WARN_ABOUT_ARM_NEON_ASM_MIT TRUE)
      endif()
    endif()

  elseif(MSVC_VERSION GREATER 1500)
    # TODO: SDL_cpuinfo.h needs to support the user's configuration wish
    # for MSVC - right now it is always activated
    if(NOT ARCH_64)
      set(HAVE_MMX TRUE)
      set(HAVE_3DNOW TRUE)
    endif()
    set(HAVE_SSE TRUE)
    set(HAVE_SSE2 TRUE)
    set(HAVE_SSE3 TRUE)
    set(SDL_ASSEMBLY_ROUTINES 1)
  endif()
# TODO:
#else()
#  if(USE_GCC OR USE_CLANG)
#    list(APPEND EXTRA_CFLAGS "-mno-sse" "-mno-sse2" "-mno-sse3" "-mno-mmx")
#  endif()
endif()

# TODO: Can't deactivate on FreeBSD? w/o LIBC, SDL_stdinc.h can't define
# anything.
if(LIBC)
  if(WINDOWS AND NOT MINGW)
    set(HAVE_LIBC TRUE)
    foreach(_HEADER stdio.h string.h wchar.h ctype.h math.h limits.h)
      string(TOUPPER "HAVE_${_HEADER}" _UPPER)
      string(REPLACE "." "_" _HAVE_H ${_UPPER})
      set(${_HAVE_H} 1)
    endforeach()
    set(HAVE_SIGNAL_H 1)
    foreach(_FN
            malloc calloc realloc free qsort abs memset memcpy memmove memcmp
            wcslen wcsdup wcsstr wcscmp wcsncmp _wcsicmp _wcsnicmp
            strlen _strrev _strupr _strlwr strchr strrchr strstr itoa _ltoa
            _ultoa strtol strtoul strtoll strtod atoi atof strcmp strncmp
            _stricmp _strnicmp strtok_s sscanf
            acos acosf asin asinf atan atanf atan2 atan2f ceil ceilf
            copysign copysignf cos cosf exp expf fabs fabsf floor floorf fmod fmodf
            log logf log10 log10f pow powf scalbn scalbnf sin sinf sqrt sqrtf tan tanf
            trunc truncf)
      string(TOUPPER ${_FN} _UPPER)
      set(HAVE_${_UPPER} 1)
    endforeach()
    if(NOT CYGWIN AND NOT MINGW)
      set(HAVE_ALLOCA 1)
    endif()
    set(HAVE_M_PI 1)
    add_definitions(-D_USE_MATH_DEFINES) # needed for M_PI
    set(STDC_HEADERS 1)
  else()
    set(HAVE_LIBC TRUE)
    check_include_file(sys/types.h HAVE_SYS_TYPES_H)
    foreach(_HEADER
            stdio.h stdlib.h stddef.h stdarg.h malloc.h memory.h string.h limits.h
            strings.h wchar.h inttypes.h stdint.h ctype.h math.h iconv.h signal.h libunwind.h)
      string(TOUPPER "HAVE_${_HEADER}" _UPPER)
      string(REPLACE "." "_" _HAVE_H ${_UPPER})
      check_include_file("${_HEADER}" ${_HAVE_H})
    endforeach()

    set(STDC_HEADER_NAMES "stddef.h;stdarg.h;stdlib.h;string.h;stdio.h;wchar.h;float.h")
    check_include_files("${STDC_HEADER_NAMES}" STDC_HEADERS)
    check_type_size("size_t" SIZEOF_SIZE_T)
    check_symbol_exists(M_PI math.h HAVE_M_PI)
    # TODO: refine the mprotect check
    check_c_source_compiles("#include <sys/types.h>
                             #include <sys/mman.h>
                             int main() { }" HAVE_MPROTECT)
    foreach(_FN
            strtod malloc calloc realloc free getenv setenv putenv unsetenv
            qsort abs bcopy memset memcpy memmove memcmp strlen strlcpy strlcat
            _strrev _strupr _strlwr strchr strrchr strstr strtok_r itoa _ltoa
            _uitoa _ultoa strtol strtoul _i64toa _ui64toa strtoll strtoull
            atoi atof strcmp strncmp _stricmp strcasecmp _strnicmp strncasecmp
            wcscmp wcsdup wcslcat wcslcpy wcslen wcsncmp wcsstr
            wcscasecmp _wcsicmp wcsncasecmp _wcsnicmp
            sscanf vsscanf vsnprintf fopen64 fseeko fseeko64 _Exit
            )
      string(TOUPPER ${_FN} _UPPER)
      set(_HAVEVAR "HAVE_${_UPPER}")
      check_symbol_exists("${_FN}" "${STDC_HEADER_NAMES}" ${_HAVEVAR})
    endforeach()

    check_symbol_exists(sigaction "signal.h" HAVE_SIGACTION)
    check_symbol_exists(setjmp "setjmp.h" HAVE_SETJMP)
    check_symbol_exists(nanosleep "time.h" HAVE_NANOSLEEP)
    check_symbol_exists(sysconf "unistd.h" HAVE_SYSCONF)
    check_symbol_exists(sysctlbyname "sys/types.h;sys/sysctl.h" HAVE_SYSCTLBYNAME)
    check_symbol_exists(getauxval "sys/auxv.h" HAVE_GETAUXVAL)
    check_symbol_exists(elf_aux_info "sys/auxv.h" HAVE_ELF_AUX_INFO)
    check_symbol_exists(poll "poll.h" HAVE_POLL)

    check_library_exists(m pow "" HAVE_LIBM)
    if(HAVE_LIBM)
      set(CMAKE_REQUIRED_LIBRARIES m)
      foreach(_FN
              atan atan2 atanf atan2f ceil ceilf copysign copysignf cos cosf
              exp expf fabs fabsf floor floorf fmod fmodf log logf log10 log10f
              pow powf scalbn scalbnf sin sinf sqrt sqrtf tan tanf acos acosf
              asin asinf trunc truncf)
        string(TOUPPER ${_FN} _UPPER)
        set(_HAVEVAR "HAVE_${_UPPER}")
        check_symbol_exists("${_FN}" "math.h" ${_HAVEVAR})
      endforeach()
      set(CMAKE_REQUIRED_LIBRARIES)
      list(APPEND EXTRA_LIBS m)
    endif()

    check_library_exists(iconv iconv_open "" HAVE_LIBICONV)
    check_library_exists(c iconv_open "" HAVE_BUILTIN_ICONV)
    if(HAVE_BUILTIN_ICONV)
      set(HAVE_ICONV 1)
    elseif(HAVE_LIBICONV)
      list(APPEND EXTRA_LIBS iconv)
      set(HAVE_ICONV 1)
    endif()

    if(NOT APPLE)
      check_include_file(alloca.h HAVE_ALLOCA_H)
      check_symbol_exists(alloca "alloca.h" HAVE_ALLOCA1)
      check_symbol_exists(alloca "stdlib.h" HAVE_ALLOCA2)
      check_symbol_exists(alloca "malloc.h" HAVE_ALLOCA3)
      if(HAVE_ALLOCA1 OR HAVE_ALLOCA2 OR HAVE_ALLOCA3)
        set(HAVE_ALLOCA 1)
      endif()
    else()
      set(HAVE_ALLOCA_H 1)
      set(HAVE_ALLOCA 1)
    endif()

    check_struct_has_member("struct sigaction" "sa_sigaction" "signal.h" HAVE_SA_SIGACTION)
  endif()
else()
  if(WINDOWS)
    set(HAVE_STDARG_H 1)
    set(HAVE_STDDEF_H 1)
  endif()
endif()


# Enable/disable various subsystems of the SDL library
foreach(_SUB ${SDL_SUBSYSTEMS})
  string(TOUPPER ${_SUB} _OPT)
  if(NOT SDL_${_OPT})
    set(SDL_${_OPT}_DISABLED 1)
  endif()
endforeach()
if(SDL_JOYSTICK)
  file(GLOB JOYSTICK_SOURCES ${SDL2_SOURCE_DIR}/src/joystick/*.c)
  set(SOURCE_FILES ${SOURCE_FILES} ${JOYSTICK_SOURCES})
endif()
if(SDL_HAPTIC)
  if(NOT SDL_JOYSTICK)
    # Haptic requires some private functions from the joystick subsystem.
    message_error("SDL_HAPTIC requires SDL_JOYSTICK, which is not enabled")
  endif()
endif()

configure_file("${SDL2_SOURCE_DIR}/include/SDL_config.h.cmake"
  "${SDL2_BINARY_DIR}/include/SDL_config.h")

# Prepare the flags and remove duplicates
if(EXTRA_LDFLAGS)
  list(REMOVE_DUPLICATES EXTRA_LDFLAGS)
endif()
if(EXTRA_LIBS)
  list(REMOVE_DUPLICATES EXTRA_LIBS)
endif()
if(EXTRA_CFLAGS)
  list(REMOVE_DUPLICATES EXTRA_CFLAGS)
endif()
listtostr(EXTRA_CFLAGS _EXTRA_CFLAGS)
set(EXTRA_CFLAGS ${_EXTRA_CFLAGS})

# Compat helpers for the configuration files
if(NOT CMAKE_HOST_WIN32)
  # TODO: we need a Windows script, too
  execute_process(COMMAND sh ${SDL2_SOURCE_DIR}/build-scripts/updaterev.sh
    WORKING_DIRECTORY ${SDL2_BINARY_DIR})
endif()
if(NOT WINDOWS OR CYGWIN)
  set(prefix ${CMAKE_INSTALL_PREFIX})
  set(exec_prefix "\${prefix}")
  set(libdir "\${exec_prefix}/lib${LIB_SUFFIX}")
  set(bindir "\${exec_prefix}/bin")
  set(includedir "\${prefix}/include")
  if(SDL_STATIC)
    set(ENABLE_STATIC_TRUE "")
    set(ENABLE_STATIC_FALSE "#")
  else()
    set(ENABLE_STATIC_TRUE "#")
    set(ENABLE_STATIC_FALSE "")
  endif()
  if(SDL_SHARED)
    set(PKG_CONFIG_LIBS_PRIV "
Libs.private:")
    set(ENABLE_SHARED_TRUE "")
    set(ENABLE_SHARED_FALSE "#")
  else()
    set(PKG_CONFIG_LIBS_PRIV "")
    set(ENABLE_SHARED_TRUE "#")
    set(ENABLE_SHARED_FALSE "")
  endif()

  # Clean up the different lists
  listtostr(EXTRA_LIBS _EXTRA_LIBS "-l")
  set(SDL_STATIC_LIBS ${EXTRA_LDFLAGS} ${_EXTRA_LIBS})
  list(REMOVE_DUPLICATES SDL_STATIC_LIBS)
  listtostr(SDL_STATIC_LIBS _SDL_STATIC_LIBS)
  set(SDL_STATIC_LIBS ${_SDL_STATIC_LIBS})
  listtostr(SDL_LIBS _SDL_LIBS)
  set(SDL_LIBS ${_SDL_LIBS})

  # MESSAGE(STATUS "SDL_LIBS: ${SDL_LIBS}")
  # MESSAGE(STATUS "SDL_STATIC_LIBS: ${SDL_STATIC_LIBS}")

  configure_file("${SDL2_SOURCE_DIR}/sdl2.pc.in"
    "${SDL2_BINARY_DIR}/sdl2.pc" @ONLY)
  configure_file("${SDL2_SOURCE_DIR}/sdl2-config.in"
    "${SDL2_BINARY_DIR}/sdl2-config")
  configure_file("${SDL2_SOURCE_DIR}/sdl2-config.in"
    "${SDL2_BINARY_DIR}/sdl2-config" @ONLY)
  configure_file("${SDL2_SOURCE_DIR}/SDL2.spec.in"
    "${SDL2_BINARY_DIR}/SDL2.spec" @ONLY)
endif()

##### Info output #####
message(STATUS "")
message(STATUS "SDL2 was configured with the following options:")
message(STATUS "")
message(STATUS "Platform: ${CMAKE_SYSTEM}")
message(STATUS "64-bit:   ${ARCH_64}")
message(STATUS "Compiler: ${CMAKE_C_COMPILER}")
message(STATUS "")
message(STATUS "Subsystems:")
foreach(_SUB ${SDL_SUBSYSTEMS})
  string(TOUPPER ${_SUB} _OPT)
  message_bool_option(${_SUB} SDL_${_OPT})
endforeach()
message(STATUS "")
message(STATUS "Options:")
list(SORT ALLOPTIONS)
foreach(_OPT ${ALLOPTIONS})
  # Longest option is VIDEO_X11_XSCREENSAVER = 22 characters
  # Get the padding
  string(LENGTH ${_OPT} _OPTLEN)
  math(EXPR _PADLEN "23 - ${_OPTLEN}")
  string(RANDOM LENGTH ${_PADLEN} ALPHABET " " _PADDING)
  message_tested_option(${_OPT} ${_PADDING})
endforeach()
message(STATUS "")
message(STATUS " CFLAGS:        ${CMAKE_C_FLAGS}")
message(STATUS " EXTRA_CFLAGS:  ${EXTRA_CFLAGS}")
message(STATUS " EXTRA_LDFLAGS: ${EXTRA_LDFLAGS}")
message(STATUS " EXTRA_LIBS:    ${EXTRA_LIBS}")
message(STATUS "")
message(STATUS " Build Shared Library: ${SDL_SHARED}")
message(STATUS " Build Static Library: ${SDL_STATIC}")
if(SDL_STATIC)
    message(STATUS " Build Static Library with Position Independent Code: ${SDL_STATIC_PIC}")
endif()
message(STATUS "")
if(UNIX)
  message(STATUS "If something was not detected, although the libraries")
  message(STATUS "were installed, then make sure you have set the")
  message(STATUS "CFLAGS and LDFLAGS environment variables correctly.")
  message(STATUS "")
endif()

if(WARN_ABOUT_ARM_SIMD_ASM_MIT)
  message(STATUS "")
  message(STATUS "SDL is being built with ARM SIMD optimizations, which")
  message(STATUS "uses code licensed under the MIT license. If this is a")
  message(STATUS "problem, please disable that code by rerunning CMake with:")
  message(STATUS "")
  message(STATUS "    -DARMSIMD=OFF")
endif()

if(WARN_ABOUT_ARM_NEON_ASM_MIT)
  message(STATUS "")
  message(STATUS "SDL is being built with ARM NEON optimizations, which")
  message(STATUS "uses code licensed under the MIT license. If this is a")
  message(STATUS "problem, please disable that code by rerunning CMake with:")
  message(STATUS "")
  message(STATUS "    -DARMNEON=OFF")
endif()

# Ensure that the extra cflags are used at compile time
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${EXTRA_CFLAGS}")

# # Always build SDLmain
# if(NOT WINDOWS_STORE)
# add_library(SDL2main STATIC ${SDLMAIN_SOURCES})
# target_include_directories(SDL2main PUBLIC "$<BUILD_INTERFACE:${SDL2_SOURCE_DIR}/include>" $<INSTALL_INTERFACE:include> $<INSTALL_INTERFACE:include/SDL2>)
# set(_INSTALL_LIBS "SDL2main")
# if (NOT ANDROID)
#   set_target_properties(SDL2main PROPERTIES DEBUG_POSTFIX "${SDL_CMAKE_DEBUG_POSTFIX}")
# endif()
# endif()

# if (ANDROID AND HAVE_HIDAPI)
#   set(_INSTALL_LIBS ${_INSTALL_LIBS} "hidapi")
# endif()

# if(SDL_SHARED)
#   add_library(SDL2 SHARED ${SOURCE_FILES} ${VERSION_SOURCES})
#   if(APPLE)
#     set_target_properties(SDL2 PROPERTIES
#       MACOSX_RPATH 1
#       OUTPUT_NAME "SDL2-${LT_RELEASE}")
#   elseif(UNIX AND NOT ANDROID)
#     set_target_properties(SDL2 PROPERTIES
#       VERSION ${LT_VERSION}
#       SOVERSION ${LT_REVISION}
#       OUTPUT_NAME "SDL2-${LT_RELEASE}")
#   else()
#     if(WINDOWS OR CYGWIN)
#       set_target_properties(SDL2 PROPERTIES
#         DEFINE_SYMBOL DLL_EXPORT)
#     endif()
#     set_target_properties(SDL2 PROPERTIES
#       VERSION ${SDL_VERSION}
#       SOVERSION ${LT_REVISION}
#       OUTPUT_NAME "SDL2")
#   endif()
#   if(MSVC AND NOT LIBC)
#     # Don't try to link with the default set of libraries.
#     set_target_properties(SDL2 PROPERTIES LINK_FLAGS_RELEASE "/NODEFAULTLIB")
#     set_target_properties(SDL2 PROPERTIES LINK_FLAGS_DEBUG "/NODEFAULTLIB")
#     set_target_properties(SDL2 PROPERTIES STATIC_LIBRARY_FLAGS "/NODEFAULTLIB")
#   endif()
#   set(_INSTALL_LIBS "SDL2" ${_INSTALL_LIBS})
#   target_link_libraries(SDL2 PRIVATE ${EXTRA_LIBS} ${EXTRA_LDFLAGS})
#   target_include_directories(SDL2 PUBLIC "$<BUILD_INTERFACE:${SDL2_SOURCE_DIR}/include>" $<INSTALL_INTERFACE:include> $<INSTALL_INTERFACE:include/SDL2>)
#   if(ANDROID)
#     target_include_directories(SDL2 PRIVATE ${ANDROID_NDK}/sources/android/cpufeatures)
#   else()
#     set_target_properties(SDL2 PROPERTIES DEBUG_POSTFIX "${SDL_CMAKE_DEBUG_POSTFIX}")
#   endif()
#   if(IOS OR TVOS)
#     set_property(TARGET SDL2 APPEND_STRING PROPERTY COMPILE_FLAGS "-fobjc-arc")
#     target_compile_definitions(SDL2 PRIVATE IOS_DYLIB=1)
#   endif()
# endif()

# if(ANDROID)
#   if(HAVE_HIDAPI)
#     add_library(hidapi SHARED ${SDL2_SOURCE_DIR}/src/hidapi/android/hid.cpp)
#   endif()

#   if(MSVC AND NOT LIBC)
#     # Don't try to link with the default set of libraries.
#     set_target_properties(hidapi PROPERTIES LINK_FLAGS_RELEASE "/NODEFAULTLIB")
#     set_target_properties(hidapi PROPERTIES LINK_FLAGS_DEBUG "/NODEFAULTLIB")
#     set_target_properties(hidapi PROPERTIES STATIC_LIBRARY_FLAGS "/NODEFAULTLIB")
#   endif()
#   if(HAVE_HIDAPI)
#     target_link_libraries(hidapi log)
#   endif()
# endif()

# if(SDL_STATIC)
#   set (BUILD_SHARED_LIBS FALSE)
#   add_library(SDL2-static STATIC ${SOURCE_FILES})
#   if (NOT SDL_SHARED OR NOT WIN32)
#     set_target_properties(SDL2-static PROPERTIES OUTPUT_NAME "SDL2")
#     # Note: Apparently, OUTPUT_NAME must really be unique; even when
#     # CMAKE_IMPORT_LIBRARY_SUFFIX or the like are given. Otherwise
#     # the static build may race with the import lib and one will get
#     # clobbered, when the suffix is realized via subsequent rename.
#   endif()
#   set_target_properties(SDL2-static PROPERTIES POSITION_INDEPENDENT_CODE ${SDL_STATIC_PIC})
#   if(MSVC AND NOT LIBC)
#     set_target_properties(SDL2-static PROPERTIES LINK_FLAGS_RELEASE "/NODEFAULTLIB")
#     set_target_properties(SDL2-static PROPERTIES LINK_FLAGS_DEBUG "/NODEFAULTLIB")
#     set_target_properties(SDL2-static PROPERTIES STATIC_LIBRARY_FLAGS "/NODEFAULTLIB")
#   endif()
#   # TODO: Win32 platforms keep the same suffix .lib for import and static
#   # libraries - do we need to consider this?
#   set(_INSTALL_LIBS "SDL2-static" ${_INSTALL_LIBS})
#   target_link_libraries(SDL2-static ${EXTRA_LIBS} ${EXTRA_LDFLAGS})
#   target_include_directories(SDL2-static PUBLIC "$<BUILD_INTERFACE:${SDL2_SOURCE_DIR}/include>" $<INSTALL_INTERFACE:include> $<INSTALL_INTERFACE:include/SDL2>)
#   if(ANDROID)
#     target_include_directories(SDL2-static PRIVATE ${ANDROID_NDK}/sources/android/cpufeatures)
#   else()
#     set_target_properties(SDL2-static PROPERTIES DEBUG_POSTFIX "${SDL_CMAKE_DEBUG_POSTFIX}")
#   endif()
#   if(IOS OR TVOS)
#     set_property(TARGET SDL2-static APPEND_STRING PROPERTY COMPILE_FLAGS "-fobjc-arc")
#   endif()
# endif()

##### Tests #####

#if(SDL_TEST)
  file(GLOB TEST_SOURCES ${SDL2_SOURCE_DIR}/src/*.c)
  add_library(SDL2_test STATIC ${TEST_SOURCES})
#endif()

##### Installation targets #####
install(TARGETS ${_INSTALL_LIBS} EXPORT SDL2Targets
  LIBRARY DESTINATION "lib${LIB_SUFFIX}"
  ARCHIVE DESTINATION "lib${LIB_SUFFIX}"
  RUNTIME DESTINATION bin)

##### Export files #####
if (WINDOWS)
  set(PKG_PREFIX "cmake")
else ()
  set(PKG_PREFIX "lib${LIB_SUFFIX}/cmake/SDL2")
endif ()

include(CMakePackageConfigHelpers)
write_basic_package_version_file("${CMAKE_BINARY_DIR}/SDL2ConfigVersion.cmake"
  VERSION ${SDL_VERSION}
  COMPATIBILITY AnyNewerVersion
)

install(EXPORT SDL2Targets
  FILE SDL2Targets.cmake
  NAMESPACE SDL2::
  DESTINATION ${PKG_PREFIX}
)
install(
  FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/SDL2Config.cmake
    ${CMAKE_BINARY_DIR}/SDL2ConfigVersion.cmake
  DESTINATION ${PKG_PREFIX}
  COMPONENT Devel
)

file(GLOB INCLUDE_FILES ${SDL2_SOURCE_DIR}/include/*.h)
file(GLOB BIN_INCLUDE_FILES ${SDL2_BINARY_DIR}/include/*.h)
foreach(_FNAME ${BIN_INCLUDE_FILES})
  get_filename_component(_INCNAME ${_FNAME} NAME)
  list(REMOVE_ITEM INCLUDE_FILES ${SDL2_SOURCE_DIR}/include/${_INCNAME})
endforeach()
list(APPEND INCLUDE_FILES ${BIN_INCLUDE_FILES})
install(FILES ${INCLUDE_FILES} DESTINATION include/SDL2)

string(TOUPPER "${CMAKE_BUILD_TYPE}" UPPER_BUILD_TYPE)
if (UPPER_BUILD_TYPE MATCHES DEBUG)
  set(SOPOSTFIX "${SDL_CMAKE_DEBUG_POSTFIX}")
else()
  set(SOPOSTFIX "")
endif()

if(NOT (WINDOWS OR CYGWIN))
  if(SDL_SHARED)
    set(SOEXT ${CMAKE_SHARED_LIBRARY_SUFFIX}) # ".so", ".dylib", etc.
    get_target_property(SONAME SDL2 OUTPUT_NAME)
    if(NOT ANDROID)
        install(CODE "
          execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink
            \"lib${SONAME}${SOPOSTFIX}${SOEXT}\" \"libSDL2${SOPOSTFIX}${SOEXT}\"
            WORKING_DIRECTORY \"${SDL2_BINARY_DIR}\")")
        install(FILES ${SDL2_BINARY_DIR}/libSDL2${SOPOSTFIX}${SOEXT} DESTINATION "lib${LIB_SUFFIX}")
    endif()
  endif()
  if(FREEBSD)
    # FreeBSD uses ${PREFIX}/libdata/pkgconfig
    install(FILES ${SDL2_BINARY_DIR}/sdl2.pc DESTINATION "libdata/pkgconfig")
  else()
    install(FILES ${SDL2_BINARY_DIR}/sdl2.pc
      DESTINATION "lib${LIB_SUFFIX}/pkgconfig")
  endif()
  install(PROGRAMS ${SDL2_BINARY_DIR}/sdl2-config DESTINATION bin)
  # TODO: what about the .spec file? Is it only needed for RPM creation?
  install(FILES "${SDL2_SOURCE_DIR}/sdl2.m4" DESTINATION "${CMAKE_INSTALL_FULL_DATAROOTDIR}/aclocal")
endif()

##### Uninstall target #####

if(NOT TARGET uninstall)
  configure_file(
      "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
      "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
      IMMEDIATE @ONLY)

  add_custom_target(uninstall
      COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
endif()
