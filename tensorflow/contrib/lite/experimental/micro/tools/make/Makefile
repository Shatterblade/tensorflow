MAKEFILE_DIR := tensorflow/contrib/lite/experimental/micro/tools/make

# Try to figure out the host system
HOST_OS :=
ifeq ($(OS),Windows_NT)
	HOST_OS = windows
else
	UNAME_S := $(shell uname -s)
	ifeq ($(UNAME_S),Linux)
		HOST_OS := linux
	endif
	ifeq ($(UNAME_S),Darwin)
		HOST_OS := osx
	endif
endif

HOST_ARCH := $(shell if [[ $(shell uname -m) =~ i[345678]86 ]]; then echo x86_32; else echo $(shell uname -m); fi)

# Override these on the make command line to target a specific architecture. For example:
# make -f tensorflow/contrib/lite/Makefile TARGET=rpi TARGET_ARCH=armv7l
TARGET := $(HOST_OS)
TARGET_ARCH := $(HOST_ARCH)

INCLUDES := \
-I. \
-I$(MAKEFILE_DIR)/../../../../../ \
-I$(MAKEFILE_DIR)/../../../../../../ \
-I$(MAKEFILE_DIR)/downloads/ \
-I$(MAKEFILE_DIR)/downloads/gemmlowp \
-I$(MAKEFILE_DIR)/downloads/flatbuffers/include \
-I$(OBJDIR)
# This is at the end so any globally-installed frameworks like protobuf don't
# override local versions in the source tree.
INCLUDES += -I/usr/local/include

TEST_SCRIPT := tensorflow/contrib/lite/experimental/micro/testing/test_linux_binary.sh

MICROLITE_LIBS := -lm

# There are no rules for compiling objects for the host system (since we don't
# generate things like the protobuf compiler that require that), so all of
# these settings are for the target compiler.
CXXFLAGS := -O3 -DNDEBUG
CXXFLAGS += --std=c++11 -g -DTF_LITE_STATIC_MEMORY
CCFLAGS := -DNDEBUG -g -DTF_LITE_STATIC_MEMORY
LDOPTS := -L/usr/local/lib
ARFLAGS := -r
TARGET_TOOLCHAIN_PREFIX :=
CC_PREFIX :=

# This library is the main target for this makefile. It will contain a minimal
# runtime that can be linked in to other programs.
MICROLITE_LIB_NAME := libtensorflow-microlite.a

# Test binary for the microcontroller speech model.
MICRO_SPEECH_TEST_SRCS := \
tensorflow/contrib/lite/experimental/micro/examples/micro_speech/micro_speech_test.cc \
tensorflow/contrib/lite/experimental/micro/examples/micro_speech/tiny_conv_model_data.cc

MICROLITE_TEST_SRCS := \
$(wildcard tensorflow/contrib/lite/experimental/micro/*test.cc) \
$(wildcard tensorflow/contrib/lite/experimental/micro/kernels/*test.cc)

MICROLITE_CC_BASE_SRCS := \
$(wildcard tensorflow/contrib/lite/experimental/micro/*.cc) \
$(wildcard tensorflow/contrib/lite/experimental/micro/kernels/*.cc) \
tensorflow/contrib/lite/c/c_api_internal.c \
tensorflow/contrib/lite/core/api/error_reporter.cc \
tensorflow/contrib/lite/core/api/flatbuffer_conversions.cc \
tensorflow/contrib/lite/core/api/op_resolver.cc \
tensorflow/contrib/lite/kernels/kernel_util.cc \
tensorflow/contrib/lite/kernels/internal/quantization_util.cc
MICROLITE_CC_SRCS := $(filter-out $(MICROLITE_TEST_SRCS), $(MICROLITE_CC_BASE_SRCS))

# These target-specific makefiles should modify or replace options like
# CXXFLAGS or LIBS to work for a specific targetted architecture. All logic
# based on platforms or architectures should happen within these files, to
# keep this main makefile focused on the sources and dependencies.
include $(wildcard $(MAKEFILE_DIR)/targets/*_makefile.inc)

ALL_SRCS := \
	$(MICRO_SPEECH_TEST_SRCS) \
	$(MICROLITE_CC_SRCS) \
	$(MICROLITE_TEST_SRCS)

# Where compiled objects are stored.
GENDIR := $(MAKEFILE_DIR)/gen/$(TARGET)_$(TARGET_ARCH)/
OBJDIR := $(GENDIR)obj/
BINDIR := $(GENDIR)bin/
LIBDIR := $(GENDIR)lib/

MICROLITE_LIB_PATH := $(LIBDIR)$(MICROLITE_LIB_NAME)

MICRO_SPEECH_TEST_BINARY := $(BINDIR)micro_speech_test

CXX := $(CC_PREFIX)${TARGET_TOOLCHAIN_PREFIX}g++
CC := $(CC_PREFIX)${TARGET_TOOLCHAIN_PREFIX}gcc
AR := $(CC_PREFIX)${TARGET_TOOLCHAIN_PREFIX}ar

MICRO_SPEECH_TEST_OBJS := $(addprefix $(OBJDIR), \
$(patsubst %.cc,%.o,$(patsubst %.c,%.o,$(MICRO_SPEECH_TEST_SRCS))))

MICROLITE_LIB_OBJS := $(addprefix $(OBJDIR), \
$(patsubst %.cc,%.o,$(patsubst %.c,%.o,$(MICROLITE_CC_SRCS))))

MICROLITE_TEST_TARGETS := $(addprefix $(BINDIR), \
$(patsubst %_test.cc,%.test_target,$(MICROLITE_TEST_SRCS)))

# For normal manually-created TensorFlow C++ source files.
$(OBJDIR)%.o: %.cc
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(INCLUDES) -c $< -o $@

# For normal manually-created TensorFlow C source files.
$(OBJDIR)%.o: %.c
	@mkdir -p $(dir $@)
	$(CC) $(CCFLAGS) $(INCLUDES) -c $< -o $@

# The target that's compiled if there's no command-line arguments.
all: $(MICROLITE_LIB_PATH) $(MICRO_SPEECH_TEST_BINARY)

microlite: $(MICROLITE_LIB_PATH)

# Hack for generating schema file bypassing flatbuffer parsing
tensorflow/contrib/lite/schema/schema_generated.h:
	@cp -u tensorflow/contrib/lite/schema/schema_generated.h.OPENSOURCE tensorflow/contrib/lite/schema/schema_generated.h

# Gathers together all the objects we've compiled into a single '.a' archive.
$(MICROLITE_LIB_PATH): tensorflow/contrib/lite/schema/schema_generated.h $(MICROLITE_LIB_OBJS)
	@mkdir -p $(dir $@)
	$(AR) $(ARFLAGS) $(MICROLITE_LIB_PATH) $(MICROLITE_LIB_OBJS)

$(MICRO_SPEECH_TEST_BINARY): $(MICRO_SPEECH_TEST_OBJS) $(MICROLITE_LIB_PATH)
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(INCLUDES) \
	-o $(MICRO_SPEECH_TEST_BINARY) $(MICRO_SPEECH_TEST_OBJS) \
	$(LIBFLAGS) $(MICROLITE_LIB_PATH) $(LDFLAGS) $(MICROLITE_LIBS)

micro_speech_test: $(MICRO_SPEECH_TEST_BINARY)
micro_speech_test_bin: $(MICRO_SPEECH_TEST_BINARY).bin

test_micro_speech: $(MICRO_SPEECH_TEST_BINARY)
	$(TEST_SCRIPT) $(MICRO_SPEECH_TEST_BINARY) '~~~ALL TESTS PASSED~~~'

$(BINDIR)%_test : $(OBJDIR)%_test.o $(MICROLITE_LIB_PATH)
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(INCLUDES) \
	-o $@ $< \
	$(LIBFLAGS) $(MICROLITE_LIB_PATH) $(LDFLAGS) $(MICROLITE_LIBS)

$(BINDIR)%.test_target: $(BINDIR)%_test
	$(TEST_SCRIPT) $< '~~~ALL TESTS PASSED~~~'

$(info $(MICROLITE_TEST_TARGETS))

test: test_micro_speech $(MICROLITE_TEST_TARGETS)

# Gets rid of all generated files.
clean:
	rm -rf $(MAKEFILE_DIR)/gen

$(DEPDIR)/%.d: ;
.PRECIOUS: $(DEPDIR)/%.d
.PRECIOUS: $(BINDIR)%_test

-include $(patsubst %,$(DEPDIR)/%.d,$(basename $(ALL_SRCS)))
