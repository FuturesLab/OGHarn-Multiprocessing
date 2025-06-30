# Compiler & sanitiser 
CC_FUZZ        = afl-clang-fast
CXX_FUZZ       = afl-clang-fast++
CFLAGS_ASAN    = -fsanitize=address,undefined
CXXFLAGS_ASAN  = -fsanitize=address,undefined

# per-process output variables
OUT      ?= $(PWD)/out/gen          # parent script may override
HARNESS  ?= harness                 # can also be overridden
TMPFILE  ?= $(OUT)/tempfile         # where showmap stores the bitmap

# Library under test
LIB_NAME  = lib
LIB_REPO  = https://github.com/DaveGamble/cJSON.git

DEPS      = -I $(PWD)/$(LIB_NAME)/ -L $(PWD)/$(LIB_NAME)/
DEPS_DYN  = -lcjson
DEPS_LDD  = $(PWD)/$(LIB_NAME)/
DEPS_STC  = -l:libcjson.a

# -------------------------------------------------------------------------
# Build the instrumented library for dynamic harness generation
lib:
	@export AFL_USE_ASAN=1;  \
	 export AFL_USE_UBSAN=1; \
	 rm -rf $(LIB_NAME);     \
	 git clone $(LIB_REPO) $(LIB_NAME); \
	 cd $(LIB_NAME) && \
	   git checkout 12c4bf1986c288950a3d06da757109a6aa1ece38 && \
	   cmake . \
	     -DCMAKE_C_COMPILER=$(CC_FUZZ) \
	     -DCMAKE_CXX_COMPILER=$(CXX_FUZZ) \
	     -DCMAKE_EXPORT_COMPILE_COMMANDS=ON && \
	   make all -j12

# Build a static copy for final fuzzing
lib_fuzz:
	@rm -rf $(LIB_NAME)_fuzz; \
	 git clone $(LIB_REPO) $(LIB_NAME)_fuzz; \
	 cd $(LIB_NAME)_fuzz && \
	   git checkout 12c4bf1986c288950a3d06da757109a6aa1ece38 && \
	   cmake . \
	     -DCMAKE_C_COMPILER=$(CC_FUZZ) \
	     -DCMAKE_CXX_COMPILER=$(CXX_FUZZ) \
	     -DBUILD_SHARED_LIBS=Off && \
	   make all -j12

# Index the library with Multiplier
run_mx:
	mx-index --db $(PWD)/$(LIB_NAME).db \
	         --target $(PWD)/$(LIB_NAME)/compile_commands.json \
	         --workspace $(PWD)/mx

# Convenience combo target
all: lib lib_fuzz run_mx

# -------------------------------------------------------------------------
# Targets used during harness generation
harness:		                    # compile the harness (dynamic)
	$(CC_FUZZ) -o $(OUT)/$(HARNESS).out \
	           $(OUT)/$(HARNESS).c \
	           $(DEPS) $(DEPS_DYN) $(CFLAGS_ASAN)

showmap:		                    # run and dump the bitmap
	LD_LIBRARY_PATH=$(DEPS_LDD) \
	afl-showmap -o $(TMPFILE) -- $(OUT)/$(HARNESS).out $(SEED)

# Same targets, but static versions (used by --execute_both)
harness_static:
	$(CC_FUZZ) -o $(OUT)/$(HARNESS).out \
	           $(OUT)/$(HARNESS).c \
	           $(DEPS) $(DEPS_DYN) -static

showmap_static:
	LD_LIBRARY_PATH=$(DEPS_LDD) \
	afl-showmap -o $(TMPFILE) -- $(OUT)/$(HARNESS).out $(SEED)

# Compile one of the final, minimised harnesses for fuzzing
harness_fuzz:		               # make harness_fuzz HARNESS_NUMBER=N
	@ls bin 2>/dev/null || mkdir bin
	$(eval LIB_NAME=lib_fuzz)
	$(CC_FUZZ) -o bin/ogharn$(HARNESS_NUMBER)_fuzz \
	           $(OUT)/final-harnesses/src/harness$(HARNESS_NUMBER):*.c \
	           -static $(DEPS) $(DEPS_STC)

# -------------------------------------------------------------------------
clean:
	rm -rf bin mx lib.db-*

clean_lib:
	rm -rf $(LIB_NAME)*
