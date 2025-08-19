# exFAT 분석
https://learn.microsoft.com/ko-kr/windows/win32/fileio/exfat-specification
https://github.com/exfatprogs/exfatprogs
https://github.com/torvalds/linux/tree/master/fs/exfat

이 곳에서 구현되어 있는 exFAT 코드를 분석하고, 이해를 해보자..

# Microsoft exFAT specification
|   이름 | 오프셋(sector) | 크기(sector) | 링크 |
|-------|---------------|-------------|-----|
| **Main Boot Region** |  |  |   |
| Main Boot Sector | 0 | 1 | [1](#main-and-backup-boot-sector-sub-regions) |
| Main Extended Boot Sectors | 1 | 8 |    |
| Main OEM Parameters | 9 | 1 |    |
| Main Reserved | 10 | 1 |    |
| Main Boot Checksum | 9 | 1 |    |
| **Backup Boot Region** |  |  |   |
| Backup Boot Sector | 12 | 1 |    |
| Backup Extended Boot Sectors | 13 | 8 |    |
| Backup OEM Parameters | 21 | 1 |    |
| Backup Reserved | 22 | 1 |    |
| Backup Boot Checksum | 23 | 1 |    |
| **FAT Region** |  |  |   |
| FAT Alignment | 24 | `FatOffset - 24` |    |
| First FAT | `FatOffset` | `FatLength` |    |
| Second FAT | `FatOffset + FatLength` | `FatLength * (NumberOfFats - 1)` |    |
| **Data Region** |  |  |   |
| Cluster Heap Alignment | `FatOffset + FatLength * NumberOfFats` | `ClusterHeapOffset-(FatOffset + FatLEngth * NumberOfFats)` |    |
| Cluster Heap | `ClusterHeapOffset` | `ClusterCount * 2^SectorsPerClusterShift` |    |
| Excess Space | `ClusterHeapOffset + ClusterCount * 2^SectorsPerClusterShift` | `VolumeLength-(ClusterHeapOffset+ClusterCount * 2^SectorsPerClusterShift)` |    |

## Main and Backup Boot Region
- Main Boot Region: 다음을 수행할 수 있도록 필요한 `boot-strapping instructions`, `identifying information`, `file system parameter`들을 제공함

1. exFAT 볼륨에서 컴퓨터 시스템을 **Boot-strap**
2. 볼륨의 파일 시스템을 `exFAT`으로 식별
3. exFAT 파일 시스템 structure들의 위치 검색

- Backup Boot Region: Main Boot Region의 백업. 필요 시 exFAT 볼륨의 복구를 진행하기에 가급적 수정해서는 안됨

### Main and Backup Boot Sector Sub-regions
- Main Boot Sector: exFAT 볼륨의 boot-strapping 코드와 볼륨 구조를 나타내는 필수 파라미터들을 포함
- Backup Boot Sector: Main Boot Sector의 백업으로 같은 구조를 갖고 있으며 보통 복구 지원용. 

이 섹터에 있는 데이터 쓰기 전에 구현에서는 각각의 **Boot Checksum** 값이 유효한지 확인하고, 각 필드가 유효나 값 범위 안에 있는지 확인해야 함

처음 포맷 과정에서 두 섹터의 값들이 초기화되긴 하지만, 필요한 경우 구현에서 이 섹터의 값들을 업데이트할 수 있음(대응되는 Boot Checksum도 업데이트 필요함). 하지만 `VolumeFlags`랑 `PercentInUse` 필드는 계산에서 제외

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| JumpBoot | 0 | 3 |
| FileSystemName | 3 | 8 |
| MustBeZero | 11 | 53 |
| PartitionOffset | 64 | 8 |
| VolumeLength | 72 | 8 |
| FatOffset | 80 | 4 |
| FatLength | 84 | 4 |
| ClusterHeapOffset | 88 | 4 |
| ClusterCount | 92 | 4 |
| FirstClusterOfRootDirectory | 96 | 4 |
| VolumeSerialNumber | 100 | 4 |
| FileSystemRevision | 104 | 2 |
| VolumeFlags | 106 | 2 |
| BytesPerSectorShift | 108 | 1 |
| SectorsPerClusterShift | 109 | 1 |
| NumberOfFats | 110 | 1 |
| DriveSelect | 111 | 1 |
| PercentInUse | 112 | 1 |
| Reserved | 113 | 7 |
| BootCode | 120 | 390 |
| BootSignature | 510 | 2 |
| ExcessSpace | 512 | 2BytesPerSectorShift – 512 |

#### JumpBoot Field
CPU의 jump instruction 포함 => 실행 되었을 때 **BootCode** 필드의 부트 스트래핑 코드로 **점프**

유효한 값은 EB 76 90

#### FileSystemName Field
볼륨의 파일 시스템 이름 포함

유효한 값은 ASCII 문자로 "EXFAT " (3개의 trailing 공백 포함)

#### MustBeZero Field
FAT12/16/32 볼륨의 packed BIOS parameter block 범위와 직접 대응

유효한 값은 0 (FAT12/16/32 구현이 실수로 exFAT 볼륨을 마운트하는 것을 방지)

#### PartitionOffset Field
exFAT 볼륨을 호스팅하는 파티션의 미디어 상대 섹터 오프셋

모든 값이 유효하지만, 0은 구현체가 이 필드를 무시하도록 함

#### VolumeLength Field
exFAT 볼륨의 크기를 섹터 단위로 표현

유효한 범위: 최소 2<sup>20</sup>/2<sup>BytesPerSectorShift</sup> (최소 1MB), 최대 2<sup>64</sup>-1

하지만 `Excess Space sub-region` 크기가 0이라면 이 필드의 최댓값은 ClusterHeapOffset + (2<sup>32</sup>-11)*2<sup>SectorsPerClusterShift</sup>

#### FatOffset Field
First FAT의 볼륨 상대 섹터 오프셋. 구현에서 기본 스토리지 미디어의 특성에 First FAT을 맞출 수 있게 해줌

유효한 범위: 최소 24, 최대 ClusterHeapOffset - (FatLength * NumberOfFats)

#### FatLength Field
각 FAT 테이블의 길이를 섹터 단위로 표현

유효한 범위: 최소 (ClusterCount + 2) * 2<sup>2</sup>/2<sup>BytesPerSectorShift</sup> (반올림), FAT가 Cluster Heap의 모든 클러스터 나타내기에 충분한 공간 있는 거 보장

최대 (ClusterHeapOffset - FatOffset) / NumberOfFats (반올림) => FAT가 Cluster Heap 전에 있는거 보장

#### ClusterHeapOffset Field
Cluster Heap의 볼륨 상대 섹터 오프셋

유효한 범위: 최소 FatOffset + FatLength * NumberOfFats, 최대 2^32-1 또는 계산된 값 중 작은 값

#### ClusterCount Field
Cluster Heap에 포함된 클러스터 수

유효한 값: 계산된 두 값 중 작은 값
1. (VolumeLength - ClusterHeapOffset) / 2<sup>SectorsPerClusterShift</sup> (반올림). 클러스터 힙의 시작 부분과 볼륨 끝 사이에 fit 가능한 클러스터 수
2. 2<sup>32</sup> - 11. FAT이 나타낼 수 있는 클러스터 최대 개수

이 필드는 FAT의 최소 크기를 결정함. 너무 큰 FAT를 막기 위해 구현에서 `SectorsPerClusterShift 필드`를 통해 클러스터 사이즈를 증가시켜 Cluster Heap의 클러스터 개수를 조절할 수 있음. 

#### FirstClusterOfRootDirectory Field
루트 디렉토리의 첫 번째 클러스터 인덱스. 루트 디렉토리는 항상 active FAT에서 클러스터 체인으로 기술되며, `GeneralPrimaryFlags` 필드의 NoFatChain 플래그가 0인 디렉토리 엔트리처럼 처리됨. 루트 디렉토리의 데이터 길이는 항상 클러스터 체인을 로딩하여 결정. 구현체는 Allocation Bitmap과 Up-case Table이 사용하는 클러스터 다음의 첫 번째 정상 클러스터에 루트 디렉토리의 첫 번째 클러스터를 배치하도록 노력해야 함

유효한 범위: 최소 2 (첫 번째 클러스터 인덱스), 최대 ClusterCount + 1(Cluster Heap의 마지막 클러스터 인덱스)

#### VolumeSerialNumber Field
고유한 시리얼 번호 포함 (서로 다른 exFAT 볼륨 구별용).구현에서는 exFAT 볼륨의 서식을 지정하는 날짜와 시간을 결합하여 일련 번호를 생성해야 함. 

모든 값이 유효 (포맷 시 날짜와 시간 조합으로 생성 권장)

#### FileSystemRevision Field
exFAT 구조체의 major/minor 리비전 번호

상위 바이트: 주 리비전, 하위 바이트: 부 리비전 (현재 스펙은 1.00)

#### VolumeFlags Field
exFAT 볼륨의 다양한 파일 시스템 구조체 상태를 나타내는 플래그. 체크섬 계산 시 제외, Backup Boot Sector에서는 stale로 처리

| 필드 이름 | 오프셋(비트) | 크기(비트) |
|-------|---------------|-------------|
| ActiveFat | 0 | 1 |
| VolumeDirty | 1 | 1 |
| MediaFailure | 2 | 1 |
| ClearToZero | 3 | 1 |
| Reserved | 4 | 12 |

##### ActiveFat
이 필드는 다음과 같이 활성 상태인 FAT와 Allocation Bitmap을 나타내야 함
- 0: First FAT과 First Allocation Bitmap이 active
- 1: Second FAT와 Second Allocation Bitmap이 active하고 `NumberOfFats` 필드에 2가 있는 경우에만 가능

##### VolumeDirty
다음과 같이 볼륨이 dirty인지 여부를 나타냄
- 0: 볼륨이 아마 consistent state
- 1: 볼륨이 아마 inconsistent state

확인되지 않는 파일 시스템 메타데이터 불일치가 발생할 때 이 필드의 값을 1로 설정 => 볼륨 탑재할 때 이 필드의 값이 1이면 메타데이터 불일치를 해결한 후에 0으로 지울 수 있음

##### MediaFailure
다음과 같이 구현에서 미디어 오류를 발견했는지 여부를 나타냄
- 0: 호스팅 미디어가 오류를 보고하지 않았거나 알려진 오류가 FAT에 이미 "잘못된" 클러스터로 기록
- 1: 호스팅 미디어가 실패 보고(예를 들어 읽기, 쓰기 실패)

다음 경우 필드 값을 1로 설정
1. 호스팅 미디어가 볼륨의 모든 지역에 대한 액세스 시도 실패
2. 구현에 액세스 다시 시도 알고리즘이 모두 사용

볼륨 탑재할 때 이 필드의 값이 1 => 전체 볼륨에서 미디어 오류 검사하고 모든 오류를 FAT의 **잘못된** 클러스터로 기록하는 구현은 0으로 지울 수 있음

##### ClearToZero
큰 의미 없음

#### BytesPerSectorShift Field
섹터당 바이트 수를 log2(N)으로 표현. N은 섹터별 바이트 개수

유효한 범위: 최소 9 (512바이트), 최대 12 (4096바이트)

#### SectorsPerClusterShift Field
클러스터당 섹터 수를 log2(N)으로 표현. N은 클러스터별 섹터 수

유효한 범위: 최소 0 (1섹터), 최대 25 - BytesPerSectorShift (32MB 클러스터)

#### NumberOfFats Field
볼륨에 포함된 FAT 및 Allocation Bitmap 수. 

유효한 값: 1 (볼륨이 First FAT랑 First Allocation Bitmap만 포함) 또는 2 (First, Second 모두 - TexFAT 볼륨에만 유효)

#### DriveSelect Field
확장 INT 13h 드라이브 번호 포함 (부트스트래핑용)

모든 값이 유효 (이전 FAT 기반 시스템에서는 주로 80h 사용)

#### PercentInUse Field
Cluster Heap에서 할당된 클러스터의 백분율

유효한 범위: 0-100 (할당된 클러스터 백분율) 또는 FFh (사용 불가능)

Cluster Heap에 있는 클러스터 할당에 변화가 생길 경우 이 필드에 반영해줘야 함. 체크섬 계산할 때는 제외

#### BootCode Field
부트스트래핑 명령어 포함

부트스트래핑 명령어 또는 F4h (halt 명령어)로 초기화

#### BootSignature Field
해당 섹터가 Boot Sector인지 여부 표시

유효한 값은 AA55h (다른 값은 Boot Sector를 무효화)

### Main and Backup Extended Boot Sectors Sub-regions
Main Extended Boot Sectors의 각 섹터는 구조가 같으나, 각 섹터가 고유한 `boot-strapping instruction`을 갖고 있을 수 있음.

Main Boot Sector의 boot-strapping instruction, alternate BIOS 구현, 임베디드 시스템의 펌웨어 같은 부트 스트래핑 에이전트들은 이러한 섹터를 로드하고 포함된 instruction을 실행할 수 있음

Main이나 Backup Extended Boot Sector의 instruction 실행 하기 전에 각 섹터의 **ExtendedBootSignature** 필드가 유효한 값 가지고 잇는지 확인해야 함

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| ExtendedBootCode | 0 | 2<sup>BytesPerSectorShift</sup>-4 |
| Extended Boot | 2<sup>BytesPerSectorShift</sup>-4 | 4 |

- 참고: Main, Backup Boot Sector 모두 **BytesPerSectorShift** 필드 갖고 있음

#### ExtendedBootCode
부트 스트래핑 instruction이나 00h(부트 스트랩 코드 제공 안할 때)

#### ExtendedBootSignature
Vaild value는 **AA550000h**. 구현에서는 Extended Boot Sector의 다른 필드로 뭘 하기 전에 반드시 이 필드 먼저 확인해야 함

### Main and Backup OEM Parameters Sub-regions
- Main OEM Parameters sub-region => 10개의 parameter structure(제조사-specific 정보 포함) 갖오 있음. 

각 파라미터들은 **Generic Parameter template** 으로부터 도출됨. 

Backup OEM Parameters는 Main OEM parameters의 백업으로 구조가 같음. 얘네로 뭐 하기 전에 각각의 Boot Checksum 확인해줘야 함. 아래는 **OEM Parameters Structure** 

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| Parameters[0] | 0 | 48 |
| . | . | . |
| . | . | . |
| . | . | . |
| Parameters[9] | 432 | 48 |
| Reserved | 480 | 2<sup>BytesPerSectorShift</sup>-480 |

#### Parameters[0] ... Parameters[9]
이 배열의 각 파라미터 필드는 **parameter structure** 을 포함함 => `Generic Parameters template`에서 도출됨. 

사용되지 않은 필드는 Null Parameter structure을 갖고 있느 걸로 나타냄

#### Generic Parameters Template
| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| ParametersGuid | 0 | 16 |
| CustomDefined | 16 | 32 |

##### ParametersGuid Field
