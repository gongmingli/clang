set(LLVM_LINK_COMPONENTS ${LLVM_TARGETS_TO_BUILD} FuzzMutate)
set(CXX_FLAGS_NOFUZZ ${CMAKE_CXX_FLAGS})
set(DUMMY_MAIN DummyClangFuzzer.cpp)
if(LLVM_LIB_FUZZING_ENGINE)
  unset(DUMMY_MAIN)
elseif(LLVM_USE_SANITIZE_COVERAGE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=fuzzer")
  set(CXX_FLAGS_NOFUZZ "${CXX_FLAGS_NOFUZZ} -fsanitize=fuzzer-no-link")
  unset(DUMMY_MAIN)
endif()

# Hack to bypass LLVM's cmake sources check and allow multiple libraries and
# executables from this directory.
set(LLVM_OPTIONAL_SOURCES
  ClangFuzzer.cpp
  DummyClangFuzzer.cpp
  ExampleClangProtoFuzzer.cpp
  )

if(CLANG_ENABLE_PROTO_FUZZER)
  # Create protobuf .h and .cc files, and put them in a library for use by
  # clang-proto-fuzzer components.
  find_package(Protobuf REQUIRED)
  add_definitions(-DGOOGLE_PROTOBUF_NO_RTTI)
  include_directories(${PROTOBUF_INCLUDE_DIRS})
  include_directories(${CMAKE_CURRENT_BINARY_DIR})
  protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS cxx_proto.proto)
  set(LLVM_OPTIONAL_SOURCES ${LLVM_OPTIONAL_SOURCES} ${PROTO_SRCS})
  add_clang_library(clangCXXProto
    ${PROTO_SRCS}
    ${PROTO_HDRS}

    LINK_LIBS
    ${PROTOBUF_LIBRARIES}
    )

  # Build and include libprotobuf-mutator
  include(ProtobufMutator)
  include_directories(${ProtobufMutator_INCLUDE_DIRS})

  # Build the protobuf->C++ translation library and driver.
  add_clang_subdirectory(proto-to-cxx)

  # Build the protobuf fuzzer
  add_clang_executable(clang-proto-fuzzer
    ${DUMMY_MAIN}
    ExampleClangProtoFuzzer.cpp
    )

  target_link_libraries(clang-proto-fuzzer
    ${ProtobufMutator_LIBRARIES}
    ${PROTOBUF_LIBRARIES}
    ${LLVM_LIB_FUZZING_ENGINE}
    clangCXXProto
    clangHandleCXX
    clangProtoToCXX
    )
endif()

add_clang_subdirectory(handle-cxx)

add_clang_executable(clang-fuzzer
  EXCLUDE_FROM_ALL
  ${DUMMY_MAIN}
  ClangFuzzer.cpp
  )

target_link_libraries(clang-fuzzer
  ${LLVM_LIB_FUZZING_ENGINE}
  clangHandleCXX
  )
