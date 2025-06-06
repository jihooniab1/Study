# LKL study
LKL 공부, 분석

# Index
- [1. Build](#build)
- [2. Example Code Analyze](#example-code-analyze)

# Build
빌드에 필요한 패키지를 설치합니다. 
```
sudo apt-get update
sudo apt-get install build-essential git gcc-multilib autoconf libtool flex bison libncurses-dev
```

소스코드를 받아줍니다.
```
git clone https://github.com/lkl/linux.git lkl
cd lkl
```

다음과 같은 커맨드로 LKL을 빌드할 수 있습니다.
```
make -C tools/lkl
```
tools/lkl/include: 이 경로에는 **lkl.h(LKL 메인 API 헤더)**, **lkl_host.h(호스트 환경 인터페이스)** 와 같은 헤더파일들이 들어있습니다.

tools/lkl/lib: 이 경로는 LKL 라이브러리의 구현체들이 빌드된 결과물이 있는 곳으로, `liblkl.so`가 LKL 최종 결과물 라이브러리입니다.

`tools/lkl/Makefile`을 살펴보면 더 자세한 빌드 흐름을 확인할 수 있습니다.
```
$(OUTPUT)Makefile.conf $(OUTPUT)include/kernel_config.h $(OUTPUT)/kernel.config: Makefile.autoconf
	$(call QUIET_AUTOCONF, headers)$(MAKE) -f Makefile.autoconf -s
```
이 부분에서 Makefile.autoconf가 호스트 환경을 감지해서 설정을 생성합니다. `LKL_HOST_CONFIG_POSIX=y` 같은 설정이 결정됩니다.

```
$(OUTPUT)lib/lkl.o: bin/stat $(ASM_CONFIG) $(DOT_CONFIG)
	$(MAKE) -C ../.. ARCH=lkl $(KOPT)
	$(MAKE) -C ../.. ARCH=lkl $(KOPT) install INSTALL_PATH=$(OUTPUT)
```
`../..`는 커널의 루트 디렉토리입니다. **ARCH=lkl**을 명시하여 LKL 아키텍처용 커널을 빌드하고 결과물로 **lib/lkl.o**가 생성됩니다.

```
$(OUTPUT)liblkl.a: $(OUTPUT)lib/liblkl-in.o $(OUTPUT)lib/lkl.o
	$(AR) -rc $@ $^

$(OUTPUT)liblkl$(SOSUF): $(OUTPUT)%-in.o $(OUTPUT)lib/lkl.o
```
`liblkl-in.o`는 LKL 구현체들을 의미하고, `lkl.o`는 리눅스 커널입니다. 둘을 합쳐서 `liblil.a, liblkl.so`를 생성합니다.

# Example Code analyze
예시로 주어진 테스트 프로그램들을 분석해봅시다

## tests/boot
LKL 초기화 과정, 커널 부팅 시퀀스, API 사용법과 같은 기본적인 LKL 사용 패턴을 보여주는 코드입니다.

main 함수는 테스트 프레임워크를 초기화하고 테스트를 실행하는 역할을 하고 있습니다.

```
int main(int argc, const char **argv)
{
	int ret;

	lkl_host_ops.print = lkl_test_log;

	if (lkl_init(&lkl_host_ops) < 0) {
		printf("%s\n", lkl_test_get_log());
		return 1;
	}

	ret = lkl_test_run(tests, sizeof(tests)/sizeof(struct lkl_test),
			"boot");

	lkl_cleanup();

	return ret;
}
```

**lkl_init** 함수에 대한 구현은 `arch/lkl/kernel`에서 찾을 수 있습니다. 다른 LKL 함수들 이전에 불려야 하는 함수입니다.
```
int lkl_init(struct lkl_host_operations *ops)
{
	if ((IS_ENABLED(CONFIG_LKL_HOST_MEMCPY) && !ops->memcpy)
	 || (IS_ENABLED(CONFIG_LKL_HOST_MEMSET) && !ops->memset)
	 || (IS_ENABLED(CONFIG_LKL_HOST_MEMMOVE) && !ops->memmove)) {
		lkl_printf("unexpected NULL lkl_host_ops member\n");
		return -1;
	}

	lkl_ops = ops;

	return kasan_init();
}
```

설정에 따라 필요한 메모리 함수들 (**memcpy, memset, memmove**)가 제공되었는지 확인하고 포인터를 검증합니다. 그 후 **lkl_ops** 전역 변수에 호스트 연산 구조체를 저장한 다음 kasan을 초기화합니다.

`lkl_host_operations`는 리눅스 커널이 사용하는 호스트 연산들입니다. 호스트 라이브러리나 애플리케이션에 의해 제공이 되어야 합니다.
```
struct lkl_host_operations {
	const char *virtio_devices;

	void (*print)(const char *str, int len);
	void (*panic)(void);

	struct lkl_sem* (*sem_alloc)(int count);
	void (*sem_free)(struct lkl_sem *sem);
	void (*sem_up)(struct lkl_sem *sem);
	void (*sem_down)(struct lkl_sem *sem);

	struct lkl_mutex *(*mutex_alloc)(int recursive);
	void (*mutex_free)(struct lkl_mutex *mutex);
	void (*mutex_lock)(struct lkl_mutex *mutex);
	void (*mutex_unlock)(struct lkl_mutex *mutex);

	lkl_thread_t (*thread_create)(void (*f)(void *), void *arg);
	void (*thread_detach)(void);
	void (*thread_exit)(void);
	int (*thread_join)(lkl_thread_t tid);
	lkl_thread_t (*thread_self)(void);
	int (*thread_equal)(lkl_thread_t a, lkl_thread_t b);
	void *(*thread_stack)(unsigned long *size);

	struct lkl_tls_key *(*tls_alloc)(void (*destructor)(void *));
	void (*tls_free)(struct lkl_tls_key *key);
	int (*tls_set)(struct lkl_tls_key *key, void *data);
	void *(*tls_get)(struct lkl_tls_key *key);

	void* (*mem_alloc)(unsigned long);
	void (*mem_free)(void *);
	void* (*page_alloc)(unsigned long size);
	void (*page_free)(void *addr, unsigned long size);

	unsigned long long (*time)(void);

	void* (*timer_alloc)(void (*fn)(void));
	int (*timer_set_oneshot)(void *timer, unsigned long delta);
	void (*timer_free)(void *timer);

	void* (*ioremap)(long addr, int size);
	int (*iomem_access)(const __volatile__ void *addr, void *val, int size,
			    int write);

	void (*jmp_buf_set)(struct lkl_jmp_buf *jmpb, void (*f)(void));
	void (*jmp_buf_longjmp)(struct lkl_jmp_buf *jmpb, int val);

	void* (*memcpy)(void *dest, const void *src, unsigned long count);
	void* (*memset)(void *s, int c, unsigned long count);
	void* (*memmove)(void *dest, const void *src, unsigned long count);

	void* (*mmap)(void *addr, unsigned long size, enum lkl_prot prot);
	int (*munmap)(void *addr, unsigned long size);

	void (*shmem_init)(unsigned long size);
	void *(*shmem_mmap)(void *addr, unsigned long pg_off, unsigned long size,
				enum lkl_prot prot);

	struct lkl_dev_pci_ops *pci_ops;
};
```
LKL은 **추상화된 가상머신**으로서 호스트가 제공하는 함수를 사용합니다. 

**lkl_ops->thread_create**를 호출해도 Linux에서는 `pthread_create`를, Windows에서는 `CreateThread`를 호출하지만 LKL은 신경쓰지 않습니다.

이제 main함수로 돌아가 **lkl_test_run** 함수를 살펴봅시다.
```
int lkl_test_run(const struct lkl_test *tests, int nr, const char *fmt, ...)
{
	int i, ret, status = TEST_SUCCESS;
	clock_t start, stop;
	char name[1024];
	va_list args;

	va_start(args, fmt);
	vsnprintf(name, sizeof(name), fmt, args);
	va_end(args);

	printf("1..%d # %s\n", nr, name);
	for (i = 1; i <= nr; i++) {
		const struct lkl_test *t = &tests[i-1];
		unsigned long delta_us;

		printf("* %d %s\n", i, t->name);
		fflush(stdout);

		start = clock();

		ret = t->fn(t->arg1, t->arg2, t->arg3);

		stop = clock();

		switch (ret) {
		case TEST_SUCCESS:
			printf("ok %d %s\n", i, t->name);
			break;
		case TEST_SKIP:
			printf("ok %d %s # SKIP\n", i, t->name);
			break;
		case TEST_BAILOUT:
			status = TEST_BAILOUT;
			/* fall through */
		case TEST_FAILURE:
		default:
			if (status != TEST_BAILOUT)
				status = TEST_FAILURE;
			printf("not ok %d %s\n", i, t->name);
		}

		printf(" ---\n");
		delta_us = (stop - start) * 1000000 / CLOCKS_PER_SEC;
		printf(" time_us: %ld\n", delta_us);
		print_log();
		printf(" ...\n");

		if (status == TEST_BAILOUT) {
			printf("Bail out!\n");
			return TEST_FAILURE;
		}

		fflush(stdout);
	}

	return status;
}
```
입력부터 살펴보면 `tests`는 테스트 배열, `nr`은 테스트 개수, `fmt,...`는 테스트 스위트 이름을 뜻합니다.

```
struct lkl_test {
	const char *name;
	int (*fn)();
	void *arg1, *arg2, *arg3;
};
```
`lkl_test` 구조체는 이렇게 정의되어 있습니다. 다양한 타입의 테스트 함수들을 통일된 방식으로 호출, 관리하는데 사용됩니다.

루프를 돌면서 테스트가 하나씩 실행되고 결과를 출력하는 방식입니다. 

```
#define LKL_TEST(name, ...) { #name, lkl_test_##name, __VA_ARGS__ }
```
이런식으로 매크로가 정의되어 있어 **lkl_test_이름** 형태의 함수를 하나씩 호출하는데 직접 살펴봅시다.

### lkl_test_mutex
```
static int lkl_test_mutex(void)
{
	long ret = TEST_SUCCESS;

	struct lkl_mutex *mutex;

	mutex = lkl_host_ops.mutex_alloc(0);
	lkl_host_ops.mutex_lock(mutex);
	lkl_host_ops.mutex_unlock(mutex);
	lkl_host_ops.mutex_free(mutex);

	mutex = lkl_host_ops.mutex_alloc(1);
	lkl_host_ops.mutex_lock(mutex);
	lkl_host_ops.mutex_lock(mutex);
	lkl_host_ops.mutex_unlock(mutex);
	lkl_host_ops.mutex_unlock(mutex);
	lkl_host_ops.mutex_free(mutex);

	return ret;
}
```

