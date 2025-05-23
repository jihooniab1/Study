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

tools/lkl/lib: 이 경로는 LKL 라이브러리의 구현체들이 빌드된 결과물이 있는 곳으로, **liblkl.so**가 LKL 최종 결과물 라이브러리입니다.

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

