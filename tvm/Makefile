ROOTDIR = $(CURDIR)

ifndef config
ifneq ("$(wildcard ./config.mk)","")
	config ?= config.mk
else
	config ?= make/config.mk
endif
endif

include $(config)

.PHONY: clean install installdev all test doc pylint cpplint lint cython cython2 cython3 runtime

ifndef DMLC_CORE_PATH
  DMLC_CORE_PATH = $(ROOTDIR)/dmlc-core
endif

ifndef DLPACK_PATH
  DLPACK_PATH = $(ROOTDIR)/dlpack
endif

UNAME_S := $(shell uname -s)

# The flags
LLVM_CFLAGS= -fno-rtti -DDMLC_ENABLE_RTTI=0 -DDMLC_USE_FOPEN64=0
LDFLAGS = -pthread -lm -ldl
INCLUDE_FLAGS = -Iinclude -I$(DLPACK_PATH)/include -I$(DMLC_CORE_PATH)/include -IHalideIR/src
CFLAGS = -std=c++11 -Wall -O2 $(INCLUDE_FLAGS) -fPIC
PKG_LDFLAGS =
FRAMEWORKS =
OBJCFLAGS = -fno-objc-arc
EMCC_FLAGS= -std=c++11 -DDMLC_LOG_STACK_TRACE=0\
	-Oz -s RESERVED_FUNCTION_POINTERS=2 -s MAIN_MODULE=1 -s NO_EXIT_RUNTIME=1\
	-s TOTAL_MEMORY=1073741824\
	-s EXTRA_EXPORTED_RUNTIME_METHODS="['cwrap','getValue','setValue','addFunction']"\
	-s USE_GLFW=3 -s USE_WEBGL2=1 -lglfw\
	$(INCLUDE_FLAGS)
# llvm configuration
ifdef LLVM_CONFIG
	LLVM_VERSION=$(shell $(LLVM_CONFIG) --version| cut -b 1,3)
	LLVM_INCLUDE=$(filter -I%, $(shell $(LLVM_CONFIG) --cxxflags))
	LDFLAGS += $(shell $(LLVM_CONFIG) --ldflags --libs --system-libs)
	LLVM_CFLAGS += $(LLVM_INCLUDE) -DTVM_LLVM_VERSION=$(LLVM_VERSION)
else
	LLVM_VERSION=00
endif

# The source code dependencies
LIB_HALIDEIR = HalideIR/lib/libHalideIR.a

CC_SRC = $(filter-out src/runtime/%.cc src/codgen/llvm/%.cc,\
             $(wildcard src/*/*.cc src/*/*/*.cc))
LLVM_SRC = $(wildcard src/codegen/llvm/*.cc src/codegen/llvm/*/*.cc)
CUDA_SRC = $(wildcard src/runtime/cuda/*.cc)
ROCM_SRC = $(wildcard src/runtime/rocm/*.cc)
OPENCL_SRC = $(wildcard src/runtime/opencl/*.cc)
OPENGL_SRC = $(wildcard src/runtime/opengl/*.cc)
VULKAN_SRC = $(wildcard src/runtime/vulkan/*.cc)
RPC_SRC = $(wildcard src/runtime/rpc/*.cc)
GRAPH_SRC = $(wildcard src/runtime/graph/*.cc)
RUNTIME_SRC = $(wildcard src/runtime/*.cc)

# Objectives
LLVM_BUILD = build/llvm${LLVM_VERSION}
LLVM_OBJ = $(patsubst src/%.cc, ${LLVM_BUILD}/%.o, $(LLVM_SRC))
CUDA_OBJ = $(patsubst src/%.cc, build/%.o, $(CUDA_SRC))
ROCM_OBJ = $(patsubst src/%.cc, build/%.o, $(ROCM_SRC))
OPENCL_OBJ = $(patsubst src/%.cc, build/%.o, $(OPENCL_SRC))
OPENGL_OBJ = $(patsubst src/%.cc, build/%.o, $(OPENGL_SRC))
VULKAN_OBJ = $(patsubst src/%.cc, build/%.o, $(VULKAN_SRC))
RPC_OBJ = $(patsubst src/%.cc, build/%.o, $(RPC_SRC))
GRAPH_OBJ = $(patsubst src/%.cc, build/%.o, $(GRAPH_SRC))
CC_OBJ = $(patsubst src/%.cc, build/%.o, $(CC_SRC)) $(LLVM_OBJ)
RUNTIME_OBJ = $(patsubst src/%.cc, build/%.o, $(RUNTIME_SRC))

# Deps
ALL_DEP = $(CC_OBJ) $(LIB_HALIDEIR)
RUNTIME_DEP = $(RUNTIME_OBJ)

ifeq ($(UNAME_S), Darwin)
	PKG_LDFLAGS += -undefined dynamic_lookup
endif

# Dependency specific rules
ifdef CUDA_PATH
	NVCC=$(CUDA_PATH)/bin/nvcc
	CFLAGS += -I$(CUDA_PATH)/include
	ifeq ($(UNAME_S),Darwin)
		LDFLAGS += -L$(CUDA_PATH)/lib
	else
		LDFLAGS += -L$(CUDA_PATH)/lib64
	endif
endif

ifeq ($(USE_CUDA), 1)
	CFLAGS += -DTVM_CUDA_RUNTIME=1
	LDFLAGS += -lcuda -lcudart -lnvrtc
	RUNTIME_DEP += $(CUDA_OBJ)
else
	CFLAGS += -DTVM_CUDA_RUNTIME=0
endif

ifdef ROCM_PATH
	CFLAGS += -I$(ROCM_PATH)/include
	LDFLAGS += -L$(ROCM_PATH)/lib
endif

ifeq ($(USE_ROCM), 1)
	CFLAGS += -DTVM_ROCM_RUNTIME=1 -D__HIP_PLATFORM_HCC__=1
	LDFLAGS += -lhip_hcc
	RUNTIME_DEP += $(ROCM_OBJ)
else
	CFLAGS += -DTVM_ROCM_RUNTIME=0
endif

ifeq ($(USE_OPENCL), 1)
	CFLAGS += -DTVM_OPENCL_RUNTIME=1
	ifeq ($(UNAME_S), Darwin)
		FRAMEWORKS += -framework OpenCL
	else
		LDFLAGS += -lOpenCL
	endif
	RUNTIME_DEP += $(OPENCL_OBJ)
else
	CFLAGS += -DTVM_OPENCL_RUNTIME=0
endif

ifeq ($(USE_VIVADO_HLS), 1)
	CFLAGS += -DHCL_VHLS_RUNTIME=1
else
	CFLAGS += -DHCL_VHLS_RUNTIME=0
endif

ifdef VULKAN_SDK
	CFLAGS += -I$(VULKAN_SDK)/include
	LDFLAGS += -L$(VULKAN_SDK)/lib
	LDFLAGS += -L$(VULKAN_SDK)/lib/spirv-tools
endif

ifeq ($(USE_VULKAN), 1)
	CFLAGS += -DTVM_VULKAN_RUNTIME=1
	LDFLAGS += -lvulkan -lSPIRV-Tools
	RUNTIME_DEP += $(VULKAN_OBJ)
else
	CFLAGS += -DTVM_VULKAN_RUNTIME=0
endif

ifeq ($(USE_OPENGL), 1)
	CFLAGS += -DTVM_OPENGL_RUNTIME=1
	EMCC_FLAGS += -DTVM_OPENGL_RUNTIME=1
	ifeq ($(UNAME_S), Darwin)
		FRAMEWORKS += -framework OpenGL
	else
		LDFLAGS += -lGL -lglfw
	endif
	RUNTIME_DEP += $(OPENGL_OBJ)
else
	CFLAGS += -DTVM_OPENGL_RUNTIME=0
endif

ifeq ($(USE_RPC), 1)
	RUNTIME_DEP += $(RPC_OBJ)
endif

ifeq ($(USE_GRAPH_RUNTIME), 1)
	RUNTIME_DEP += $(GRAPH_OBJ)
endif

ifeq ($(USE_GRAPH_RUNTIME_DEBUG), 1)
	CFLAGS += -DTVM_GRAPH_RUNTIME_DEBUG
endif

ifdef ADD_CFLAGS
	CFLAGS += $(ADD_CFLAGS)
endif

ifdef ADD_LDFLAGS
	LDFLAGS += $(ADD_LDFLAGS)
endif

ifeq ($(OS),Windows_NT)
	SHARED_LIBRARY_SUFFIX := dll
else
	UNAME_S := $(shell uname -s)
	ifeq ($(UNAME_S), Darwin)
		SHARED_LIBRARY_SUFFIX := dylib
	else
		SHARED_LIBRARY_SUFFIX := so
	endif
endif

BUILD_TARGETS ?= lib/libhcl.$(SHARED_LIBRARY_SUFFIX) \
	lib/libhcl_runtime.$(SHARED_LIBRARY_SUFFIX) \

all: ${BUILD_TARGETS}
runtime: lib/libhcl_runtime.$(SHARED_LIBRARY_SUFFIX)

include tests/cpp/unittest.mk

test: $(TEST)


# Special rules for LLVM related modules.
${LLVM_BUILD}/codegen/llvm/%.o: src/codegen/llvm/%.cc
	@mkdir -p $(@D)
	$(CXX) $(CFLAGS) $(LLVM_CFLAGS) -MM -MT ${LLVM_BUILD}/codegen/llvm/$*.o $< >${LLVM_BUILD}/codegen/llvm/$*.d
	$(CXX) -c $(CFLAGS) $(LLVM_CFLAGS) -c $< -o $@

build/%.o: src/%.cc
	@mkdir -p $(@D)
	$(CXX) $(CFLAGS) -MM -MT build/$*.o $< >build/$*.d
	$(CXX) -c $(CFLAGS) -c $< -o $@

lib/libhcl.${SHARED_LIBRARY_SUFFIX}: $(ALL_DEP) $(RUNTIME_DEP)
	@mkdir -p $(@D)
	$(CXX) $(CFLAGS) $(FRAMEWORKS) -shared -o $@ $(filter %.o %.a, $^) $(LDFLAGS)

lib/libhcl_runtime.${SHARED_LIBRARY_SUFFIX}: $(RUNTIME_DEP)
	@mkdir -p $(@D)
	$(CXX) $(CFLAGS) $(FRAMEWORKS) -shared -o $@ $(filter %.o %.a, $^) $(LDFLAGS)

$(LIB_HALIDEIR): LIBHALIDEIR

LIBHALIDEIR:
	+ cd HalideIR; make lib/libHalideIR.a DMLC_CORE_PATH=../dmlc-core; cd $(ROOTDIR)

cpplint:
	python dmlc-core/scripts/lint.py tvm cpp include src\
	 examples/extension/src examples/graph_executor/src

pylint:
	pylint python/tvm --rcfile=$(ROOTDIR)/tests/lint/pylintrc

lint: cpplint pylint

doc:
	doxygen docs/Doxyfile

install: lib/libhcl_runtime.$(SHARED_LIBRARY_SUFFIX)
	mkdir -p $(DESTDIR)$(PREFIX)/include/tvm/runtime
	cp -R include/tvm/runtime/. $(DESTDIR)$(PREFIX)/include/tvm/runtime
	cp lib/libhcl_runtime.$(SHARED_LIBRARY_SUFFIX) $(DESTDIR)$(PREFIX)/lib

installdev: lib/libhcl.$(SHARED_LIBRARY_SUFFIX) lib/libhcl_runtime.$(SHARED_LIBRARY_SUFFIX) lib/libhcl.a
	mkdir -p $(DESTDIR)$(PREFIX)/include
	cp -R include/tvm $(DESTDIR)$(PREFIX)/include
	cp lib/libhcl.$(SHARED_LIBRARY_SUFFIX) $(DESTDIR)$(PREFIX)/lib
	cp lib/libhcl_runtime.$(SHARED_LIBRARY_SUFFIX) $(DESTDIR)$(PREFIX)/lib
	cp lib/libhcl.a $(DESTDIR)$(PREFIX)/lib

# Cython build
cython:
	cd python; python setup.py build_ext --inplace

cython2:
	cd python; python2 setup.py build_ext --inplace

cython3:
	cd python; python3 setup.py build_ext --inplace

cyclean:
	rm -rf python/tvm/*/*/*.so python/tvm/*/*/*.dylib python/tvm/*/*/*.cpp

clean:
	$(RM) -rf build lib bin *~ */*~ */*/*~ */*/*/*~ */*.o */*/*.o */*/*/*.o */*.d */*/*.d */*/*/*.d
	cd HalideIR; make clean; cd $(ROOTDIR)

-include build/*.d
-include build/*/*.d
-include build/*/*/*.d
-include build/*/*/*/*.d
