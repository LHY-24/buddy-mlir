#!/bin/bash
BUDDY_OPT := ../../build/bin/buddy-opt
BUDDY_TRANSLATE := ../../build/bin/buddy-translate
MLIR_OPT := ../../llvm/build/bin/mlir-opt
MLIR_TRANSLATE := ../../llvm/build/bin/mlir-translate
MLIR_CPU_RUNNER := ../../llvm/build/bin/mlir-cpu-runner
LLC := ../../llvm/build/bin/llc
OPT_FLAG := -O0

RISCV_GNU_TOOLCHAIN := ../../thirdparty/build-riscv-gnu-toolchain
RISCV_GNU_TOOLCHAIN_SYSROOT := ../../thirdparty/build-riscv-gnu-toolchain/sysroot
QEMU := ../../thirdparty/qemu/build/riscv64-linux-user/qemu-riscv64
LOCAL_CLANG := ../../thirdparty/build-local-clang/bin/clang
CROSS_LLI := ../../thirdparty/build-cross-clang/bin/lli
CROSS_MLIR_CPU_RUNNER := ../../thirdparty/build-cross-mlir/bin/mlir-cpu-runner
CROSS_MLIR_C_RUNNER_UTILS := ../../thirdparty/build-cross-mlir/lib/libmlir_c_runner_utils.so
CROSS_MLIR_RUNNER_UTILS := ../../thirdparty/build-cross-mlir/lib/libmlir_runner_utils.so
CROSS_MLIR_LIB := ../../thirdparty/build-cross-mlir/lib

ifeq ($(shell uname),Linux)
MLIR_RUNNER_UTILS := ../../llvm/build/lib/libmlir_runner_utils.so
MLIR_C_RUNNER_UTILS := ../../llvm/build/lib/libmlir_c_runner_utils.so
MTRIPLE := x86_64-unknown-linux-gnu
else ifeq ($(shell uname),Darwin)
MLIR_RUNNER_UTILS := ../../llvm/build/lib/libmlir_runner_utils.dylib
MLIR_C_RUNNER_UTILS := ../../llvm/build/lib/libmlir_c_runner_utils.dylib
MTRIPLE := x86_64-apple-darwin
endif

vector-exp-config-lower:
	@${BUDDY_OPT} ./vector-exp-predication.mlir \
		-lower-vector-exp \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts \
		-o ./log.mlir

vector-exp-config-translate:
	@${BUDDY_OPT} ./vector-exp-predication.mlir \
		-lower-vector-exp \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts | \
	${BUDDY_TRANSLATE} --buddy-to-llvmir -o log.ll

vector-exp-config-run:
	@${BUDDY_OPT} ./vector-exp-predication.mlir \
		-lower-vector-exp \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts | \
	${MLIR_CPU_RUNNER} ${OPT_FLAG} -e main -entry-point-result=void \
		-shared-libs=${MLIR_RUNNER_UTILS} -shared-libs=${MLIR_C_RUNNER_UTILS}
	
vector-exp-predication-memory-lower:
	@${BUDDY_OPT} ./vector-exp-predication-memory.mlir \
		-lower-vector-exp \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts \
		-o ./log.mlir

vector-exp-predication-memory-run:
	@${BUDDY_OPT} ./vector-exp-predication-memory.mlir \
		-lower-vector-exp \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts |\
	${BUDDY_TRANSLATE} -buddy-to-llvmir | \
	${QEMU} -L ${RISCV_GNU_TOOLCHAIN_SYSROOT} \
			-cpu rv64,x-v=true,vlen=128 \
	${CROSS_LLI} -march=riscv64 -mattr=+m,+d,+v \
		-dlopen=${CROSS_MLIR_C_RUNNER_UTILS} \
		-dlopen=${CROSS_MLIR_RUNNER_UTILS}

vector-exp-predication-matmul-run:
	@${BUDDY_OPT} ./vector-exp-predication-matmul.mlir \
		-lower-affine \
		-convert-scf-to-cf \
		-convert-math-to-llvm \
		-lower-vector-exp \
		-lower-rvv \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts |\
	${BUDDY_TRANSLATE} -buddy-to-llvmir | \
	${QEMU} -L ${RISCV_GNU_TOOLCHAIN_SYSROOT} \
			-cpu rv64,x-v=true,vlen=128 \
	${CROSS_LLI} -march=riscv64 -mattr=+m,+d,+v \
		-dlopen=${CROSS_MLIR_C_RUNNER_UTILS} \
		-dlopen=${CROSS_MLIR_RUNNER_UTILS}

vector-exp-add-mask-run:
	@${BUDDY_OPT} ./vector-exp-add-mask.mlir \
		-lower-affine \
		-convert-scf-to-cf \
		-convert-math-to-llvm \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts | \
	${BUDDY_TRANSLATE} -buddy-to-llvmir | \
	${QEMU} -L ${RISCV_GNU_TOOLCHAIN_SYSROOT} \
			-cpu rv64,x-v=true,vlen=128 \
	${CROSS_LLI} -march=riscv64 -mattr=+m,+d,+v \
		-dlopen=${CROSS_MLIR_C_RUNNER_UTILS} \
		-dlopen=${CROSS_MLIR_RUNNER_UTILS}

vector-exp-add-mask-asm:
	@${BUDDY_OPT} ./vector-exp-add-mask.mlir \
		-lower-affine \
		-convert-scf-to-cf \
		-convert-math-to-llvm \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts | \
	${BUDDY_TRANSLATE} -buddy-to-llvmir | \
	${LLC} ${OPT_FLAG} -mtriple riscv64 -target-abi lp64d \
		-mattr=+m,+d,+v -riscv-v-vector-bits-min=128 \
		--filetype=asm -o log.s

vector-exp-add-predication-run:
	@${BUDDY_OPT} ./vector-exp-add-predication.mlir \
		-lower-affine \
		-convert-scf-to-cf \
		-convert-math-to-llvm \
		-lower-vector-exp \
		-lower-rvv \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts | \
	${BUDDY_TRANSLATE} -buddy-to-llvmir | \
	${QEMU} -L ${RISCV_GNU_TOOLCHAIN_SYSROOT} \
			-cpu rv64,x-v=true,vlen=128 \
	${CROSS_LLI} -march=riscv64 -mattr=+m,+d,+v \
		-dlopen=${CROSS_MLIR_C_RUNNER_UTILS} \
		-dlopen=${CROSS_MLIR_RUNNER_UTILS}

vector-exp-add-predication-asm:
	@${BUDDY_OPT} ./vector-exp-add-predication.mlir \
		-lower-affine \
		-convert-scf-to-cf \
		-convert-math-to-llvm \
		-lower-vector-exp \
		-lower-rvv \
		-convert-vector-to-llvm \
		-convert-memref-to-llvm \
		-convert-func-to-llvm \
		-reconcile-unrealized-casts | \
	${BUDDY_TRANSLATE} -buddy-to-llvmir | \
	${LLC} ${OPT_FLAG} -mtriple riscv64 -target-abi lp64d \
		-mattr=+m,+d,+v -riscv-v-vector-bits-min=128 \
		--filetype=asm -o log.s
