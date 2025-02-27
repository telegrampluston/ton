cmake_minimum_required(VERSION 3.0.2 FATAL_ERROR)

project(TON VERSION 0.5 LANGUAGES C CXX)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
#set(OPENSSL_USE_STATIC_LIBS TRUE)

# Define the two required variables before including the source code for watching a git repository.
set(PRE_CONFIGURE_FILE "git.cc.in")
set(POST_CONFIGURE_FILE "${CMAKE_CURRENT_BINARY_DIR}/git.cc")
include(git_watcher.cmake)

# Create a library out of the compiled post-configure file.
add_library(git STATIC ${POST_CONFIGURE_FILE})
target_include_directories(git PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
add_dependencies(git check_git)

# Prevent in-source build
get_filename_component(TON_REAL_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}" REALPATH)
get_filename_component(TON_REAL_BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}" REALPATH)

if (TON_REAL_BINARY_DIR STREQUAL TON_REAL_SOURCE_DIR)
  message("  Out-of-source build should be used to build TON.")
  message("  You need to remove the files already created by CMake and")
  message("  rerun CMake from a new directory:")
  message("  rm -rf CMakeFiles CMakeCache.txt")
  message("  mkdir build")
  message("  cd build")
  message("  cmake ..")
  message(FATAL_ERROR "In-source build failed.")
endif()

# HAVE_SSE42 for crc32c and rocksdb
include(CheckCXXSourceCompiles)
# Check for SSE4.2 support in the compiler.
set(OLD_CMAKE_REQURED_FLAGS ${CMAKE_REQUIRED_FLAGS})
if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} /arch:AVX")
else(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -msse4.2")
endif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
check_cxx_source_compiles("
#if defined(_MSC_VER)
#include <intrin.h>
#else  // !defined(_MSC_VER)
#include <cpuid.h>
#include <nmmintrin.h>
#endif  // defined(_MSC_VER)

int main() {
  _mm_crc32_u8(0, 0); _mm_crc32_u32(0, 0);
#if defined(_M_X64) || defined(__x86_64__)
   _mm_crc32_u64(0, 0);
#endif // defined(_M_X64) || defined(__x86_64__)
  return 0;
}
"  CRC32C_HAVE_SSE42)
set(CMAKE_REQUIRED_FLAGS ${OLD_CMAKE_REQURED_FLAGS})

if(NOT MSVC)
  set(CMAKE_REQUIRED_FLAGS "-msse4.2 -mpclmul")
endif()
CHECK_CXX_SOURCE_COMPILES("
#include <cstdint>
#include <nmmintrin.h>
#include <wmmintrin.h>
int main() {
  volatile uint32_t x = _mm_crc32_u32(0, 0);
  const auto a = _mm_set_epi64x(0, 0);
  const auto b = _mm_set_epi64x(0, 0);
  const auto c = _mm_clmulepi64_si128(a, b, 0x00);
  auto d = _mm_cvtsi128_si64(c);
}
" ROCKSDB_HAVE_SSE42)
unset(CMAKE_REQUIRED_FLAGS)

if (ROCKSDB_HAVE_SSE42 AND CRC32C_HAVE_SSE42)
  set(HAVE_SSE42 TRUE)
else()
  set(HAVE_SSE42 FALSE)
endif()

#BEGIN internal
option(TON_ONLY_TONLIB "Use \"ON\" to build only tonlib." OFF)
if (TON_ONLY_TONLIB)
  set(NOT_TON_ONLY_TONLIB false)
else()
  set(NOT_TON_ONLY_TONLIB true)
endif()
option(TON_USE_ROCKSDB "Use \"ON\" to enable RocksDb." ${NOT_TON_ONLY_TONLIB})
option(TON_USE_ABSEIL "Use \"ON\" to enable Abseil." ${NOT_TON_ONLY_TONLIB})
option(TON_USE_JEMALLOC "Use \"ON\" to enable JeMalloc." OFF)
#END internal

option(TONLIB_ENABLE_JNI "Use \"ON\" to enable JNI-compatible TonLib API.")
option(TON_USE_ASAN "Use \"ON\" to enable AddressSanitizer." OFF)
option(TON_USE_TSAN "Use \"ON\" to enable ThreadSanitizer." OFF)
option(TON_USE_UBSAN "Use \"ON\" to enable UndefinedBehaviorSanitizer." OFF)
set(TON_ARCH "native" CACHE STRING "Architecture, will be passed to -march=")

if (TON_USE_ABSEIL)
  message("Add abseil-cpp")
  add_subdirectory(third-party/abseil-cpp EXCLUDE_FROM_ALL)
  set(ABSL_FOUND 1)
endif()

#add_subdirectory(third-party/libcuckoo EXCLUDE_FROM_ALL)
#add_subdirectory(third-party/junction EXCLUDE_FROM_ALL)

if (WIN32)
  message("Add wingetopt")
  add_subdirectory(third-party/wingetopt EXCLUDE_FROM_ALL)
  set(WINGETOPT_FOUND 1)
  message(STATUS "Use wingetopt")
endif()

set(CRC32C_BUILD_TESTS OFF CACHE BOOL "Build CRC32C's unit tests")
set(CRC32C_BUILD_BENCHMARKS OFF CACHE BOOL "Build CRC32C's benchmarks")
set(CRC32C_USE_GLOG OFF CACHE BOOL "Build CRC32C's tests with Google Logging")
set(CRC32C_INSTALL OFF CACHE BOOL "Install CRC32C's header and library")
message("Add crc32c")
add_subdirectory(third-party/crc32c EXCLUDE_FROM_ALL)
set(CRC32C_FOUND 1)

if (TON_USE_ROCKSDB)
  if (ANDROID)
    set(PORTABLE ON CACHE BOOL "portable")
  endif()
  set(WITH_GFLAGS OFF CACHE BOOL "build with GFlags")
  set(WITH_TESTS OFF CACHE BOOL "build with tests")
  set(WITH_TOOLS OFF CACHE BOOL "build with tools")
  set(FAIL_ON_WARNINGS OFF CACHE BOOL "fail on warnings")
  message("Add rocksdb")
  add_subdirectory(third-party/rocksdb EXCLUDE_FROM_ALL)
endif()

option(USE_COROUTINES "experimental support of coroutines" OFF)
if (USE_COROUTINES)
  if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(TD_HAVE_COROUTINES 1)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcoroutines-ts")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
  endif()
endif()

option(USE_LIBRAPTORQ "use libraptorq for tests" OFF)
if (USE_LIBRAPTORQ)
  set(USE_LZ4 OFF CACHE BOOL "use lz4")
  if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(CLANG_STDLIB "ON") # for libraptorq
  endif()
  message("Add libraptorq")
  add_subdirectory(third-party/libraptorq EXCLUDE_FROM_ALL)
endif()

message("Add ton")
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMake" ${CMAKE_MODULE_PATH})

# Configure CCache if available
find_program(CCACHE_FOUND ccache)
#set(CCACHE_FOUND 0)
if (CCACHE_FOUND)
  message(STATUS "Found ccache")
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
else()
  message(STATUS "Could NOT find ccache")
endif()

if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  set(GCC 1)
elseif (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  set(CLANG 1)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "Intel")
  set(INTEL 1)
elseif (NOT MSVC)
  message(FATAL_ERROR "Compiler isn't supported")
endif()

include(CheckCXXCompilerFlag)

if (GCC OR CLANG OR INTEL)
  if (WIN32 AND INTEL)
    set(STD14_FLAG /Qstd=c++14)
  else()
    set(STD14_FLAG -std=c++14)
  endif()
  check_cxx_compiler_flag(${STD14_FLAG} HAVE_STD14)
  if (NOT HAVE_STD14)
    string(REPLACE "c++14" "c++1y" STD14_FLAG "${STD14_FLAG}")
    check_cxx_compiler_flag(${STD14_FLAG} HAVE_STD1Y)
    set(HAVE_STD14 ${HAVE_STD1Y})
  endif()
elseif (MSVC)
  set(HAVE_STD14 MSVC_VERSION>=1900)
endif()

if (NOT HAVE_STD14)
  message(FATAL_ERROR "No C++14 support in the compiler. Please upgrade the compiler.")
endif()

set(CMAKE_THREAD_PREFER_PTHREAD ON)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
find_package(ZLIB REQUIRED)

if (TON_ARCH AND NOT MSVC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=${TON_ARCH}")
endif()
if (THREADS_HAVE_PTHREAD_ARG)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
endif()

if (TON_USE_JEMALLOC)
  find_package(JeMalloc REQUIRED)
endif()

set(MEMPROF "" CACHE STRING "Use one of \"ON\", \"FAST\" or \"SAFE\" to enable memory profiling. \
Works under macOS and Linux when compiled using glibc. \
In FAST mode stack is unwinded only using frame pointers, which may fail. \
In SAFE mode stack is unwinded using backtrace function from execinfo.h, which may be very slow. \
By default both methods are used to achieve maximum speed and accuracy")

if (CLANG OR GCC)
  if (MEMPROF)
    check_cxx_compiler_flag(-no-pie CXX_NO_PIE_FLAG)
    if (CXX_NO_PIE_FLAG)
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -no-pie")
    elseif (APPLE)
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-no_pie")
    endif()
  endif()
endif()

if (MSVC)
  if (CMAKE_CXX_FLAGS_DEBUG MATCHES "/RTC1")
    string(REPLACE "/RTC1" " " CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
  endif()
  add_definitions(-D_SCL_SECURE_NO_WARNINGS -D_CRT_SECURE_NO_WARNINGS)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP /W4 /wd4100 /wd4127 /wd4324 /wd4456 /wd4457 /wd4458 /wd4505 /wd4702")
elseif (CLANG OR GCC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${STD14_FLAG} -fno-omit-frame-pointer")
  if (APPLE)
    #use "-Wl,-exported_symbols_list,${CMAKE_CURRENT_SOURCE_DIR}/export_list" for exported symbols
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fvisibility=hidden -Wl,-dead_strip,-x,-S")
    set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} -fvisibility=hidden -Wl,-dead_strip,-x,-S")
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ffunction-sections -fdata-sections")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--gc-sections -Wl,--exclude-libs,ALL")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--gc-sections")
    if (NOT TON_USE_ASAN AND NOT TON_USE_TSAN AND NOT MEMPROF)
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--exclude-libs,ALL")
    endif()
  endif()
elseif (INTEL)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${STD14_FLAG}")
endif()

if (WIN32)
  add_definitions(-DNTDDI_VERSION=0x06020000 -DWINVER=0x0602 -D_WIN32_WINNT=0x0602 -DNOMINMAX -DUNICODE -D_UNICODE)
endif()
if (CYGWIN)
  add_definitions(-D_DEFAULT_SOURCE=1 -DFD_SETSIZE=4096)
endif()

if (NOT ANDROID) # _FILE_OFFSET_BITS is broken in ndk r15 and r15b and doesn't work prior to Android 7.0
  add_definitions(-D_FILE_OFFSET_BITS=64)
endif()

set(INTERNAL_COMPILE "0")
#BEGIN internal
  add_definitions(-D_INTERNAL_COMPILE=1)
  set(INTERNAL_COMPILE "1")
#END internal

set(TONLIB_COMPILE "0")
#BEGIN tonlib
  add_definitions(-D_TONLIB_COMPILE=1)
  set(TONLIB_COMPILE "1")
#END tonlib

include(AddCXXCompilerFlag)
if (MSVC)
  add_cxx_compiler_flag("/experimental:external /external:anglebrackets /external:W0")
endif()
if (NOT MSVC)
  add_cxx_compiler_flag("-Wall")
endif()
add_cxx_compiler_flag("-Wextra")
add_cxx_compiler_flag("-Wimplicit-fallthrough=2")
add_cxx_compiler_flag("-Wpointer-arith")
add_cxx_compiler_flag("-Wcast-qual")
add_cxx_compiler_flag("-Wsign-compare")
add_cxx_compiler_flag("-Wduplicated-branches")
add_cxx_compiler_flag("-Wduplicated-cond")
add_cxx_compiler_flag("-Walloc-zero")
add_cxx_compiler_flag("-Wlogical-op")
add_cxx_compiler_flag("-Wno-tautological-compare")
add_cxx_compiler_flag("-Wpointer-arith")
add_cxx_compiler_flag("-Wvla")
add_cxx_compiler_flag("-Wnon-virtual-dtor")
add_cxx_compiler_flag("-Wno-unused-parameter")
add_cxx_compiler_flag("-Wconversion")
add_cxx_compiler_flag("-Wno-sign-conversion")
add_cxx_compiler_flag("-Qunused-arguments")
add_cxx_compiler_flag("-Wno-unused-private-field")
add_cxx_compiler_flag("-Wno-redundant-move")
#add_cxx_compiler_flag("-Werror")

#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -isystem /usr/include/c++/v1")
if (CLANG)
  #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
endif()
if (TON_USE_ASAN)
  if (CLANG)
    add_cxx_compiler_flag("-stdlib=libc++")
  endif()
  add_cxx_compiler_flag("-fsanitize=address")
  add_definitions(-DTD_USE_ASAN=1)
endif()
if (TON_USE_TSAN)
  if (CLANG)
    add_cxx_compiler_flag("-stdlib=libc++")
  endif()
  add_cxx_compiler_flag("-fsanitize=thread")
endif()
if (TON_USE_UBSAN)
  if (CLANG)
    add_cxx_compiler_flag("-stdlib=libc++")
  endif()
  add_cxx_compiler_flag("-fsanitize=undefined")
endif()
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=thread")
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined")
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=leak")
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -finstrument-functions")

#Compilation database
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

#BEGIN internal
find_package(LATEX)
if (LATEX_FOUND)
  include(UseLATEX)
  add_latex_document(doc/ton.tex TARGET_NAME ton_white_paper)
  add_latex_document(doc/tvm.tex TARGET_NAME ton_vm_description)
  add_latex_document(doc/tblkch.tex TARGET_NAME ton_blockchain_description)
  add_latex_document(doc/fiftbase.tex TARGET_NAME fift_basic_description)
  add_latex_document(doc/catchain.tex TARGET_NAME catchain_consensus_description)
endif()
#END internal

function(target_link_libraries_system target)
  set(libs ${ARGN})
  foreach(lib ${libs})
    get_target_property(lib_include_dirs ${lib} INTERFACE_INCLUDE_DIRECTORIES)
    target_include_directories(${target} SYSTEM PUBLIC ${lib_include_dirs})
    target_link_libraries(${target} PUBLIC ${lib})
  endforeach(lib)
endfunction(target_link_libraries_system)

set(TDUTILS_MIME_TYPE OFF CACHE BOOL "Generate mime type conversion")
add_subdirectory(tdutils)
add_subdirectory(memprof)
add_subdirectory(tdactor)
add_subdirectory(tdnet)
if (TON_USE_ROCKSDB)
  option(TDDB_USE_ROCKSDB "Use rockdb" ON)
endif()
add_subdirectory(tddb)
add_subdirectory(tdtl)
add_subdirectory(tl)
add_subdirectory(terminal)
add_subdirectory(keys)
add_subdirectory(tl-utils)
add_subdirectory(adnl)
add_subdirectory(crypto)
add_subdirectory(lite-client)

#BEGIN tonlib
add_subdirectory(tonlib)
#END tonlib

#BEGIN internal
if (NOT TON_ONLY_TONLIB)
add_subdirectory(common)
add_subdirectory(tdfec)
add_subdirectory(keyring)
add_subdirectory(fec)
add_subdirectory(rldp)
add_subdirectory(rldp2)
add_subdirectory(dht)
add_subdirectory(overlay)
add_subdirectory(catchain)
add_subdirectory(validator-session)
add_subdirectory(validator)
add_subdirectory(blockchain-explorer)
add_subdirectory(storage)
add_subdirectory(validator-engine)
add_subdirectory(validator-engine-console)
add_subdirectory(create-hardfork)
add_subdirectory(dht-server)
add_subdirectory(utils)
add_subdirectory(http)
add_subdirectory(rldp-http-proxy)
endif()
#END internal

if (NOT CMAKE_CROSSCOMPILING)
  if (TDUTILS_MIME_TYPE)
    set(TDMIME_AUTO tdmime_auto)
  endif()
  add_custom_target(prepare_cross_compiling DEPENDS tl_generate_common tlb_generate_block gen_fif ${TDMIME_AUTO})
endif()

#TESTS
add_executable(test-ed25519 test/test-td-main.cpp ${ED25519_TEST_SOURCE})
target_link_libraries(test-ed25519 PRIVATE ton_crypto)

add_executable(test-vm test/test-td-main.cpp ${TONVM_TEST_SOURCE})
target_link_libraries(test-vm PRIVATE ton_crypto fift-lib)

add_executable(test-smartcont test/test-td-main.cpp ${SMARTCONT_TEST_SOURCE})
target_link_libraries(test-smartcont PRIVATE smc-envelope fift-lib ton_db)

add_executable(test-cells test/test-td-main.cpp ${CELLS_TEST_SOURCE})
target_link_libraries(test-cells PRIVATE ton_crypto)

add_executable(test-fift test/test-td-main.cpp ${FIFT_TEST_SOURCE})
target_link_libraries(test-fift PRIVATE fift-lib)

add_executable(test-tdutils test/test-td-main.cpp ${TDUTILS_TEST_SOURCE})
target_link_libraries(test-tdutils PRIVATE tdutils ${CMAKE_THREAD_LIBS_INIT} memprof ${JEMALLOC_LIBRARIES})
#target_link_libraries_system(test-tdutils absl::base absl::container absl::hash )
#target_link_libraries_system(test-tdutils libcuckoo)
#target_include_directories(test-tdutils PRIVATE SYSTEM ${JUNCTION_ALL_INCLUDE_DIRS})
#target_link_libraries(test-tdutils PRIVATE ${JUNCTION_ALL_LIBRARIES})

add_executable(test-tdactor test/test-td-main.cpp ${TDACTOR_TEST_SOURCE})
target_link_libraries(test-tdactor PRIVATE tdactor ${CMAKE_THREAD_LIBS_INIT})

add_executable(test-net test/test-td-main.cpp ${NET_TEST_SOURCE})
target_link_libraries(test-net PRIVATE tdnet tdutils ${CMAKE_THREAD_LIBS_INIT})

#BEGIN tonlib
add_executable(test-tonlib ${TONLIB_ONLINE_TEST_SOURCE})
target_link_libraries(test-tonlib tdutils tdactor adnllite tl_api ton_crypto ton_block tl_tonlib_api tonlib)

add_executable(test-tonlib-offline test/test-td-main.cpp ${TONLIB_OFFLINE_TEST_SOURCE})
target_link_libraries(test-tonlib-offline tdutils tdactor adnllite tl_api ton_crypto ton_block fift-lib tl_tonlib_api tonlib)

if (NOT CMAKE_CROSSCOMPILING)
  add_dependencies(test-tonlib-offline gen_fif)
endif()
#END tonlib

#BEGIN internal
if (NOT TON_ONLY_TONLIB)
add_executable(test-db test/test-td-main.cpp ${TONDB_TEST_SOURCE})
target_link_libraries(test-db PRIVATE ton_db memprof tdfec)

add_executable(test-storage test/test-td-main.cpp ${STORAGE_TEST_SOURCE})
target_link_libraries(test-storage PRIVATE storage ton_db memprof tl_api tl-utils fec rldp2)

add_executable(test-rocksdb test/test-rocksdb.cpp)
target_link_libraries(test-rocksdb PRIVATE memprof tddb tdutils)

add_executable(test-tddb test/test-td-main.cpp ${TDDB_TEST_SOURCE})
target_link_libraries(test-tddb PRIVATE tdutils tddb ${CMAKE_THREAD_LIBS_INIT} memprof)

add_executable(test-fec test/test-td-main.cpp ${FEC_TEST_SOURCE})
target_link_libraries(test-fec PRIVATE tdfec tdutils ${CMAKE_THREAD_LIBS_INIT})
if (USE_LIBRAPTORQ)
  target_link_libraries(test-fec PRIVATE third_party_fec)
  target_compile_definitions(test-fec PRIVATE "USE_LIBRAPTORQ=1")
endif()

add_executable(test-hello-world test/test-hello-world.cpp )
target_link_libraries(test-hello-world tl_api ton_crypto)

add_executable(test-adnl test/test-adnl.cpp)
target_link_libraries(test-adnl adnl adnltest dht tl_api)
add_executable(test-dht test/test-dht.cpp)
target_link_libraries(test-dht adnl adnltest dht tl_api)
add_executable(test-rldp test/test-rldp.cpp)
target_link_libraries(test-rldp adnl adnltest dht rldp tl_api)
add_executable(test-rldp2 test/test-rldp2.cpp)
target_link_libraries(test-rldp2 adnl adnltest dht rldp2 tl_api)
add_executable(test-validator-session-state test/test-validator-session-state.cpp)
target_link_libraries(test-validator-session-state adnl dht rldp validatorsession tl_api)

#add_executable(test-node test/test-node.cpp)
#target_link_libraries(test-node overlay tdutils tdactor adnl tl_api dht
#  catchain validatorsession)

add_executable(test-catchain test/test-catchain.cpp)
target_link_libraries(test-catchain overlay tdutils tdactor adnl adnltest rldp tl_api dht
  catchain )
#add_executable(test-validator-session test/test-validator-session.cpp)
#target_link_libraries(test-validator-session overlay tdutils tdactor adnl tl_api dht
#  catchain validatorsession)
add_executable(test-ton-collator test/test-ton-collator.cpp)
target_link_libraries(test-ton-collator overlay tdutils tdactor adnl tl_api dht
  catchain validatorsession validator-disk ton_validator validator-disk )
#add_executable(test-validator test/test-validator.cpp)
#target_link_libraries(test-validator overlay tdutils tdactor adnl tl_api dht
#    rldp catchain validatorsession ton-node validator ton_validator validator memprof ${JEMALLOC_LIBRARIES})
#add_executable(test-ext-server test/test-ext-server.cpp)
#target_link_libraries(test-ext-server tdutils tdactor adnl tl_api dht )
#add_executable(test-ext-client test/test-ext-client.cpp)
#target_link_libraries(test-ext-client tdutils tdactor adnl tl_api tl-lite-utils)

add_executable(test-http test/test-http.cpp)
target_link_libraries(test-http PRIVATE tonhttp)

get_directory_property(HAS_PARENT PARENT_DIRECTORY)
if (HAS_PARENT)
  set(ALL_TEST_SOURCE
          ${TDUTILS_TEST_SOURCE}
          ${TDACTOR_TEST_SOURCE}
          ${NET_TEST_SOURCE}
          ${TDDB_TEST_SOURCE}
          ${FEC_TEST_SOURCE}
          ${ED25519_TEST_SOURCE}
          ${TONDB_TEST_SOURCE}
          ${CELLS_TEST_SOURCE} # ${TONVM_TEST_SOURCE} ${FIFT_TEST_SOURCE} ${TONLIB_ONLINE_TEST_SOURCE}
          PARENT_SCOPE)
endif()
add_library(all_tests INTERFACE)
target_link_libraries(all_tests INTERFACE tdutils tdactor tdnet tdfec ton_db ton_crypto fift-lib)
endif()
#END internal

enable_testing()
set(TEST_OPTIONS "--regression ${CMAKE_CURRENT_SOURCE_DIR}/test/regression-tests.ans --filter -Bench")
separate_arguments(TEST_OPTIONS)
add_test(test-ed25519-crypto crypto/test-ed25519-crypto)
add_test(test-ed25519 test-ed25519)
add_test(test-vm test-vm ${TEST_OPTIONS})
add_test(test-fift test-fift ${TEST_OPTIONS})
add_test(test-cells test-cells ${TEST_OPTIONS})
add_test(test-smartcont test-smartcont)
add_test(test-net test-net)
add_test(test-actors test-tdactor)

#BEGIN tonlib
add_test(test-tdutils test-tdutils)
add_test(test-tonlib-offline test-tonlib-offline)
#END tonlib

#BEGIN internal
if (NOT TON_ONLY_TONLIB)
add_test(test-adnl test-adnl)
add_test(test-dht test-dht)
add_test(test-rldp test-rldp)
add_test(test-rldp2 test-rldp2)
#add_test(test-validator-session-state test-validator-session-state)
add_test(test-catchain test-catchain)

add_test(test-fec test-fec)
add_test(test-tddb test-tddb ${TEST_OPTIONS})
add_test(test-db test-db ${TEST_OPTIONS})
endif()
#END internal

