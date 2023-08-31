OUTPUT = ./output
LIBBPF = ../libbpf

LIBBPF_SRC = $(abspath $(LIBBPF)/src)
LIBBPF_OBJ = $(abspath $(OUTPUT)/libbpf.a)

CC = gcc
CLANG = clang-14

ARCH := $(shell uname -m)
ARCH := $(subst x86_64,amd64,$(ARCH))
GOARCH := $(ARCH)

BPFTOOL = $(shell which bpftool || /bin/false)
BTFFILE = /sys/kernel/btf/vmlinux
DBGVMLINUX = /usr/lib/debug/boot/vmlinux-$(shell uname -r)
GIT = $(shell which git || /bin/false)
VMLINUXH = vmlinux.h

# libbpf


LIBBPF_OBJDIR = $(abspath ./$(OUTPUT)/libbpf)
LIBBPF_DESTDIR = $(abspath ./$(OUTPUT))

# 传递给编译器参数设置
# -ggdb：这个选项告诉编译器生成用于调试的符号信息，以便在调试程序时可以查看变量、函数等的值。
# -gdwarf：这个选项指定了用于调试信息的DWARF格式。
# -O2：这个选项启用了中等级别的优化。编译器将尝试优化生成的代码以提高性能，但仍然保留了调试信息。
# -Wall：这个选项启用了警告信息，编译器将生成与潜在问题相关的警告。这有助于开发者捕获潜在的代码问题。
# -fpie：这个选项告诉编译器生成可执行文件中的位置独立代码。这在生成可执行文件时有助于增强安全性，因为它使得攻击者更难以利用特定的内存布局。
# -Wno-unused-variable：这个选项告诉编译器忽略未使用的变量的警告。有时候，开发者可能会声明但尚未使用某些变量，这个选项可以防止编译器生成未使用变量的警告。
# -Wno-unused-function：这个选项告诉编译器忽略未使用的函数的警告。类似于 -Wno-unused-variable，它防止编译器生成未使用函数的警告。
CFLAGS = -ggdb -gdwarf -O2 -Wall -fpie -Wno-unused-variable -Wno-unused-function

# 传递给链接器参数设置
LDFLAGS =

BPF_CFLAGS_STATIC = "-I$(abspath $(OUTPUT))"
BPF_LDFLAGS_STATIC = "-lelf -lz $(LIBBPF_OBJ)"

# cgo 编译选项，指定-I头文件 地址
CGO_CFLAGS_STATIC = "-I$(abspath $(OUTPUT))"
CGO_LDFLAGS_STATIC = "-lelf -lz $(LIBBPF_OBJ)"
CGO_EXTLDFLAGS_STATIC = '-w -extldflags "-static"'

CGO_CFGLAGS_DYN = "-I. -I/usr/include/"
CGO_LDFLAGS_DYN = "-lelf -lz -lbpf"
CGO_EXTLDFLAGS_DYN = '-w'


.PHONY: $(PROGRAM)
.PHONY: $(PROGRAM).bpf.c


PROGRAM = main

# 相当于执行 make -C . main
all:
	$(MAKE) -C . $(PROGRAM)


# vmlinux header file

.PHONY: vmlinuxh
vmlinuxh: $(VMLINUXH)

$(VMLINUXH): $(OUTPUT)
ifeq ($(wildcard $(BPFTOOL)),)
	@echo "ERROR: could not find bpftool"
	@exit 1
endif
	@if [ -f $(DBGVMLINUX) ]; then \
		echo "INFO: found dbg kernel, generating $(VMLINUXH) from $(DBGVMLINUX)"; \
		$(BPFTOOL) btf dump file $(DBGVMLINUX) format c > $(VMLINUXH); \
	fi
	@if [ ! -f $(BTFFILE) ] && [ ! -f $(DBGVMLINUX) ]; then \
		echo "ERROR: kernel does not seem to support BTF"; \
		exit 1; \
	fi
	@if [ ! -f $(VMLINUXH) ]; then \
		echo "INFO: generating $(VMLINUXH) from $(BTFFILE)"; \
		$(BPFTOOL) btf dump file $(BTFFILE) format c > $(VMLINUXH); \
	fi


# static libbpf generation for the git submodule

.PHONY: libbpf
libbpf: $(LIBBPF_OBJ)


# $(LIBBPF_SRC) 如果该目录中的文件或子目录有任何变化，都会触发重新构建 $(LIBBPF_OBJ)
# wildcard 查找LIBBPF_SRC 目录中的所有 .c 和 .h 文件。如果这些源文件中的任何一个被修改，也会触发重新构建
# 目标对象的依赖关系，如A: B|C A依赖B,C   当B有变化时也需要重新编译A,但是当C变化时不强制重新编译A
# 内容是设置了一些环境变量和目录，编译了libbpf
$(LIBBPF_OBJ): $(LIBBPF_SRC) $(wildcard $(LIBBPF_SRC)/*.[ch]) | $(OUTPUT)/libbpf
	CC="$(CC)" CFLAGS="$(CFLAGS)" LD_FLAGS="$(LDFLAGS)" \
	   $(MAKE) -C $(LIBBPF_SRC) \
		BUILD_STATIC_ONLY=1 \
		OBJDIR=$(LIBBPF_OBJDIR) \
		DESTDIR=$(LIBBPF_DESTDIR) \
		INCLUDEDIR= LIBDIR= UAPIDIR= prefix= libdir= install



# 进行更新库文件。递归地更新所有子模块的内容
$(LIBBPF_SRC):
ifeq ($(wildcard $@), )
	echo "INFO: updating submodule 'libbpf'"
	$(GIT) submodule update --init --recursive
endif



$(OUTPUT):
	mkdir -p $(OUTPUT)

$(OUTPUT)/libbpf:
	mkdir -p $(OUTPUT)/libbpf



# $(PROGRAM).bpf.o: 这是目标的名称，也就是要构建的文件名。在这种情况下，它表示要构建的目标文件是 $(PROGRAM).bpf.o。
# $(PROGRAM).bpf.c: 这是目标的一个依赖项，表示要构建 $(PROGRAM).bpf.o 目标文件，需要先构建 $(PROGRAM).bpf.c 这个C源文件。 $< 是一个自动变量，表示规则中的第一个依赖项，也就是 $(PROGRAM).bpf.c。
# |: 这是一个竖线符号，用于分隔正常的依赖项和仅顺序的依赖项。在这个规则中，它表示 vmlinuxh 是仅顺序的依赖项，这意味着 vmlinuxh 会在构建 $(PROGRAM).bpf.o 之前构建，但不会强制重新构建 $(PROGRAM).bpf.o，除非 vmlinuxh 发生了变化。
# vmlinuxh: 这是一个仅顺序的依赖项，它表示 $(PROGRAM).bpf.o 目标在构建之前需要确保 vmlinuxh 文件已经存在或已经构建。仅顺序的依赖项通常用于确保某些文件或操作在目标构建之前完成，但不会强制目标重新构建。
# $(CLANG) $(CFLAGS) -target bpf -D__TARGET_ARCH_x86 -I. -I$(OUTPUT) -c $< -o $@: 这是规则的命令部分，它描述了如何构建 $(PROGRAM).bpf.o 目标。具体来说，它使用 clang 编译器和一系列选项来将 $(PROGRAM).bpf.c 编译为 $(PROGRAM).bpf.o 目标文件。$< 和 $@ 是自动变量，分别表示规则中的第一个依赖项（$(PROGRAM).bpf.c）和目标文件名（$(PROGRAM).bpf.o）。这个命令将源文件编译为目标文件。
$(PROGRAM).bpf.o: $(PROGRAM).bpf.c | vmlinuxh
	$(CLANG) $(CFLAGS) -target bpf -D__TARGET_ARCH_x86 -I. -I$(OUTPUT) -c $< -o $@


## GO example

.PHONY: $(PROGRAM)

# 真正执行cgo 编译二进制程序，依赖libbpf目标 和 $(PROGRAM).bpf.o目标，即ebpf c程序的编译结果
$(PROGRAM): libbpf | $(PROGRAM).bpf.o
	CC=$(CLANG) \
		CGO_CFLAGS=$(CGO_CFLAGS_STATIC) \
		CGO_LDFLAGS=$(CGO_LDFLAGS_STATIC) \
                GOARCH=$(GOARCH) \
                go build \
                -tags netgo -ldflags $(CGO_EXTLDFLAGS_STATIC) \
                -o $(PROGRAM) ./$(PROGRAM).go


## clean
clean:
	$(MAKE) -C $(LIBBPF_SRC) clean
	rm -rf $(OUTPUT)
	rm -rf $(VMLINUXH)
	rm -rf $(PROGRAM) $(PROGRAM)-*static $(PROGRAM)-*dynamic
	rm -rf $(PROGRAM).bpf.o $(PROGRAM).o

/tmp:
	pwd


test: /tmp
	123




