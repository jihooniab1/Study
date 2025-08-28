# exFAT 분석
https://learn.microsoft.com/ko-kr/windows/win32/fileio/exfat-specification
https://github.com/exfatprogs/exfatprogs
https://github.com/torvalds/linux/tree/master/fs/exfat

이 곳에서 구현되어 있는 exFAT 코드를 분석하고, 이해를 해보자..

# Index
- [1. Microsoft exFAT spec](#microsoft-exfat-specification)
- [2. Linux Kernel Analyze](#linux-커널-코드-분석)
    - [2.1 Inode](#아이노드)
    - [2.2 Header files](#헤더파일-분석)
        - [2.2.1 exfat_raw.h](#exfat_rawh)
        - [2.2.2 exfat_fs.h](#exfat_fsh)
    - [2.3 exFAT init, disk mount](#파일-시스템-초기화-및-디스크-마운트)

# Microsoft exFAT specification
|   이름 | 오프셋(sector) | 크기(sector) | 링크 |
|-------|---------------|-------------|-----|
| **Main Boot Region** |  |  |   |
| Main Boot Sector | 0 | 1 | [1](#main-and-backup-boot-sector-sub-regions) |
| Main Extended Boot Sectors | 1 | 8 |  [2](#main-and-backup-extended-boot-sectors-sub-regions)  |
| Main OEM Parameters | 9 | 1 |  [3](#main-and-backup-oem-parameters-sub-regions)  |
| Main Reserved | 10 | 1 |    |
| Main Boot Checksum | 11 | 1 |  [4](#main-and-backup-boot-checksum-sub-regions)  |
| **Backup Boot Region** |  |  |   |
| Backup Boot Sector | 12 | 1 |    |
| Backup Extended Boot Sectors | 13 | 8 |    |
| Backup OEM Parameters | 21 | 1 |    |
| Backup Reserved | 22 | 1 |    |
| Backup Boot Checksum | 23 | 1 |    |
| **FAT Region** |  |  |  [5](#file-allocation-table-region)  |
| FAT Alignment | 24 | FatOffset - 24 |    |
| First FAT | FatOffset | FatLength |    |
| Second FAT | FatOffset + FatLength | FatLength * (NumberOfFats - 1) |    |
| **Data Region** |  |  | [6](#data-region)  |
| Cluster Heap Alignment | FatOffset + FatLength * NumberOfFats | ClusterHeapOffset-(FatOffset + FatLEngth * NumberOfFats) |    |
| Cluster Heap | ClusterHeapOffset | ClusterCount * 2<sup>SectorsPerClusterShift</sup> |    |
| Excess Space | ClusterHeapOffset + ClusterCount * 2<sup>SectorsPerClusterShift</sup> | VolumeLength-(ClusterHeapOffset+ClusterCount * 2<sup>SectorsPerClusterShift</sup>) |    |

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

사용되지 않은 필드는 Null Parameter structure을 갖고 있는 걸로 나타냄

#### Generic Parameters Template
| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| ParametersGuid | 0 | 16 |
| CustomDefined | 16 | 32 |

##### ParametersGuid
지정된 parameter structure의 나머지 레이아웃을 결정하는 GUID를 나타냄

#### Null Parameters
Generic Parameters template에서 파생되고 `unused Parameters field`를 나타내야함. OEM Parameters structure을 만들거나 업데이트할 때 구현은 안 쓰는 매개 변수 필드를 Null Parameter structure로 채워야함. 

#### Flash Parameters
Generic Parameters template에서 파생되며 플래시 미디어에 대한 매개변수를 포함함. 플래시 기반 스토리지 장치 제조사들은 Parameters 필드(가급적 Parameters[0] 필드)를 이 파라미터 구조로 채울 수 있음. 구현체들은 Flash Parameters 구조의 정보를 사용하여 읽기/쓰기 중 접근 작업을 최적화하고 미디어 포맷팅 중 파일 시스템 구조의 정렬을 위해 사용할 수 있음.

Flash Parameters 구조에 대한 지원은 선택사항임.

**Flash Parameters Structure**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| ParametersGuid | 0 | 16 |
| EraseBlockSize | 16 | 4 |
| PageSize | 20 | 4 |
| SpareSectors | 24 | 4 |
| RandomAccessTime | 28 | 4 |
| ProgrammingTime | 32 | 4 |
| ReadCycle | 36 | 4 |
| WriteCycle | 40 | 4 |
| Reserved | 44 | 4 |

**참고:** ParametersGuid 필드를 제외한 모든 Flash Parameters 필드의 모든 가능한 값은 유효함. 하지만 값이 0이면 해당 필드가 실제로는 의미가 없음을 나타냄(구현체는 해당 필드를 무시해야 함).

##### ParametersGuid 필드
Generic Parameters template에서 제공된 정의를 따름.

이 필드의 유효한 값은 GUID 표기법으로 `{0A0C7E46-3399-4021-90C8-FA6D389C4BA2}` 임.

##### EraseBlockSize 필드
플래시 미디어의 지우기 블록 크기를 바이트 단위로 설명함.

##### PageSize 필드
플래시 미디어의 페이지 크기를 바이트 단위로 설명함.

##### SpareSectors 필드
플래시 미디어가 내부 스패어링 작업에 사용할 수 있는 섹터 수를 설명함.

##### RandomAccessTime 필드
플래시 미디어의 평균 랜덤 액세스 시간을 나노초 단위로 설명함.

##### ProgrammingTime 필드
플래시 미디어의 평균 프로그래밍 시간을 나노초 단위로 설명함.

##### ReadCycle 필드
플래시 미디어의 평균 읽기 사이클 시간을 나노초 단위로 설명함.

##### WriteCycle 필드
평균 쓰기 사이클 시간을 나노초 단위로 설명함.

### Main and Backup Boot Checksum Sub-regions
Main, Backup Boot Checksum은 각 Boot region의 다른 모든 sub-region에 대한 반복되는 패턴의 4 바이트 체크섬을 갖고 있음. 

`VolumeFlags`, `PercentInUse` 필드는 포함 X

반복되는 패턴의 4 바이트 체크섬은 **Boot Checksum sub-region** 처음부터 끝까지 채운다. Boot region의 어떤 content를 쓰기 전에 Boot checksum으로 먼저 확인을 해야 함. 

```
UInt32 BootChecksum
(
    UCHAR  * Sectors,        // points to an in-memory copy of the 11 sectors
    USHORT   BytesPerSector
)
{
    UInt32 NumberOfBytes = (UInt32)BytesPerSector * 11;
    UInt32 Checksum = 0;
    UInt32 Index;

    for (Index = 0; Index < NumberOfBytes; Index++)
    {
        if ((Index == 106) || (Index == 107) || (Index == 112))
        {
            continue;
        }
        Checksum = ((Checksum&1) ? 0x80000000 : 0) + (Checksum>>1) + (UInt32)Sectors[Index];
    }

    return Checksum;
}
```

## File Allocation Table Region
File Allocation Table (FAT) 영역은 최대 2개의 FAT를 포함할 수 있음: 하나는 First FAT sub-region에, 다른 하나는 Second FAT sub-region에 위치. NumberOfFats 필드가 이 영역에 포함된 FAT 개수를 설명함. NumberOfFats 필드의 유효한 값은 1과 2임. 따라서 First FAT sub-region은 항상 FAT를 포함하고, NumberOfFats 필드가 2라면 Second FAT sub-region도 FAT를 포함함.

VolumeFlags 필드의 ActiveFat 필드가 어떤 FAT가 활성화되어 있는지 설명함. Main Boot Sector의 VolumeFlags 필드만이 현재 값임. 구현들은 활성화되지 않은 FAT를 오래된 것으로 처리해야 함. 비활성 FAT의 사용과 FAT 간 전환은 구현에 따라 다름.

### First and Second FAT Sub-regions
FAT는 Cluster Heap의 **cluster chain** 들을 설명해야 함. 클러스터 체인은 파일, 디렉터리, 기타 파일시스템 구조의 내용을 기록하기 위한 공간을 제공하는 일련의 클러스터들임. FAT는 클러스터 체인을 클러스터 인덱스의 단일 연결 리스트로 나타냄. 처음 두 엔트리를 제외하고, FAT의 모든 엔트리는 정확히 하나의 클러스터를 나타냄.

**File Allocation Table Structure**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| FatEntry[0] | 0 | 4 |
| FatEntry[1] | 4 | 4 |
| FatEntry[2] | 8 | 4 |
| . | . | . |
| . | . | . |
| . | . | . |
| FatEntry[ClusterCount+1] | (ClusterCount + 1) * 4 | 4 |
| ExcessSpace | (ClusterCount + 2) * 4 | (FatLength * 2<sup>BytesPerSectorShift</sup>) – ((ClusterCount + 2) * 4) |

**참고:** 
- ClusterCount + 1은 절대 FFFFFFF6h를 초과할 수 없음
- Main과 Backup Boot Sector 모두 ClusterCount 필드를 포함함
- Main과 Backup Boot Sector 모두 ClusterCount, FatLength, BytesPerSectorShift 필드를 포함함

#### FatEntry[0] 필드
FatEntry[0] 필드는 첫 번째 바이트(최하위 바이트)에서 미디어 타입을 설명하고 나머지 3바이트에는 FFh를 포함해야 함.

미디어 타입(첫 번째 바이트)은 F8h여야 함.

#### FatEntry[1] 필드
FatEntry[1] 필드는 역사적 선례 때문에만 존재하며 특별한 의미를 갖지 않음.

이 필드의 유효한 값은 FFFFFFFFh임. 구현체들은 이 필드를 규정된 값으로 초기화해야 하며 어떤 목적으로도 이 필드를 사용해서는 안 됨. 구현체들은 이 필드를 해석하지 말고 주변 필드를 수정하는 작업에서도 그 내용을 보존해야 함.

#### FatEntry[2] ... FatEntry[ClusterCount+1] 필드들
이 배열의 각 FatEntry 필드는 Cluster Heap의 클러스터를 나타내야 함. FatEntry[2]는 Cluster Heap의 첫 번째 클러스터를 나타내고 FatEntry[ClusterCount+1]은 Cluster Heap의 마지막 클러스터를 나타냄.

이러한 필드들의 유효한 값 범위는 다음과 같음:

- **2부터 ClusterCount + 1까지(포함)**: 주어진 클러스터 체인의 다음 FatEntry를 가리킴; 주어진 FatEntry는 주어진 클러스터 체인에서 그 앞에 오는 어떤 FatEntry도 가리켜서는 안 됨
- **정확히 FFFFFFF7h**: 주어진 FatEntry의 해당 클러스터를 "배드"로 표시함
- **정확히 FFFFFFFFh**: 주어진 FatEntry의 해당 클러스터를 클러스터 체인의 마지막 클러스터로 표시함; 이는 주어진 클러스터 체인의 마지막 FatEntry에 대한 유일한 유효한 값임

## Data Region
Data 영역은 Cluster Heap을 포함하며, 파일시스템 구조, 디렉터리, 파일을 위한 관리된 공간을 제공함.

### Cluster Heap Sub-region
Cluster Heap의 구조는 매우 간단함; SectorsPerClusterShift 필드가 정의하는 대로 각각의 연속된 섹터 시리즈가 하나의 클러스터를 설명함. 중요한 점은 Cluster Heap의 첫 번째 클러스터가 인덱스 2를 가지며, 이는 FatEntry[2]의 인덱스와 직접적으로 대응됨.

exFAT 볼륨에서는 **Allocation Bitmap** 이 모든 클러스터의 할당 상태 기록을 유지함. 이는 Cluster Heap의 모든 클러스터 할당 상태 기록을 FAT가 유지했던 exFAT의 전신들(FAT12, FAT16, FAT32)과의 중요한 차이점임.

**Cluster Heap Structure**

| 필드 이름 | 오프셋(섹터) | 크기(섹터) |
|-------|---------------|-------------|
| Cluster[2] | ClusterHeapOffset | 2<sup>SectorsPerClusterShift</sup> |
| . | . | . |
| . | . | . |
| . | . | . |
| Cluster[ClusterCount+1] | ClusterHeapOffset + (ClusterCount – 1) * 2<sup>SectorsPerClusterShift</sup> | 2<sup>SectorsPerClusterShift</sup> |

**참고:** 
- Main과 Backup Boot Sector 모두 ClusterHeapOffset과 SectorsPerClusterShift 필드를 포함함
- Main과 Backup Boot Sector 모두 ClusterCount, ClusterHeapOffset, SectorsPerClusterShift 필드를 포함함

#### Cluster[2] ... Cluster[ClusterCount+1] 필드들
이 배열의 각 Cluster 필드는 연속된 섹터들의 시리즈이며, 그 크기는 SectorsPerClusterShift 필드에 의해 정의됨.

## Directory Structure
exFAT 파일시스템은 Cluster Heap에 존재하는 파일시스템 구조와 파일들을 관리하기 위해 **디렉터리 트리** 접근법을 사용함. 디렉터리는 디렉터리 트리에서 부모와 자식 간에 일대다 관계를 가짐.

`FirstClusterOfRootDirectory` 필드가 참조하는 디렉터리가 디렉터리 트리의 루트임. 다른 모든 디렉터리들은 단일 연결 방식으로 루트 디렉터리로부터 내려옴.

각 디렉터리는 일련의 디렉터리 엔트리들로 구성됨.

하나 이상의 디렉터리 엔트리들이 결합되어 파일시스템 구조, 하위 디렉터리, 또는 파일과 같은 관심 대상을 설명하는 디렉터리 엔트리 세트를 형성함.

**Directory Structure**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| DirectoryEntry[0] | 0 | 32 |
| . | . | . |
| . | . | . |
| . | . | . |
| DirectoryEntry[N–1] | (N – 1) * 32 | 32 |

**참고:** N(DirectoryEntry 필드의 수)은 주어진 디렉터리를 포함하는 클러스터 체인의 크기(바이트)를 DirectoryEntry 필드의 크기(32바이트)로 나눈 값임.

### DirectoryEntry[0] ... DirectoryEntry[N-1]
이 배열의 각 DirectoryEntry 필드는 Generic DirectoryEntry template에서 파생됨.

### Generic DirectoryEntry Template
Generic DirectoryEntry template은 디렉터리 엔트리들에 대한 기본 정의를 제공함. 모든 디렉터리 엔트리 구조는 이 템플릿에서 파생되며 Microsoft에서 정의한 디렉터리 엔트리 구조만이 유효함. Generic DirectoryEntry template을 해석하는 능력은 필수임.

**Generic DirectoryEntry Template**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| EntryType | 0 | 1 |
| CustomDefined | 1 | 19 |
| FirstCluster | 20 | 4 |
| DataLength | 24 | 8 |

#### EntryType 필드
EntryType 필드는 필드의 값이 정의하는 세 가지 사용 모드를 가짐:

**00h** - 디렉터리 끝 마커:
- 주어진 DirectoryEntry의 다른 모든 필드는 실제로는 reserved
- 주어진 디렉터리의 모든 후속 디렉터리 엔트리들도 디렉터리 끝 마커임
- 디렉터리 끝 마커는 디렉터리 엔트리 세트 외부에서만 유효함
- 구현체들은 필요에 따라 디렉터리 끝 마커 overwrite 가능

**01h~7Fh(포함)** - 사용되지 않는 디렉터리 엔트리 마커:
- 주어진 DirectoryEntry의 다른 모든 필드는 실제로 정의되지 않음
- 사용되지 않는 디렉터리 엔트리는 디렉터리 엔트리 세트 외부에서만 유효함
- 구현체들은 필요에 따라 사용되지 않는 디렉터리 엔트리를 덮어쓸 수 있음
- 이 값 범위는 InUse 필드가 값 0을 포함하는 것에 해당함

**81h~FFh(포함)** - 일반 디렉터리 엔트리:
- EntryType 필드의 내용이 DirectoryEntry 구조의 나머지 레이아웃을 결정함
- 이 값 범위만이 디렉터리 엔트리 세트 내부에서 유효함
- 이 값 범위는 InUse 필드가 값 1을 포함하는 것에 직접 해당함

InUse 필드의 수정이 잘못되어 디렉터리 끝 마커가 되는 것을 방지하기 위해 값 80h는 무효임.

**Generic EntryType Field Structure**

| 필드 이름 | 오프셋(비트) | 크기(비트) |
|-------|---------------|-------------|
| TypeCode | 0 | 5 |
| TypeImportance | 5 | 1 |
| TypeCategory | 6 | 1 |
| InUse | 7 | 1 |

##### TypeCode 필드
주어진 디렉터리 엔트리의 특정 타입을 부분적으로 설명함. 이 필드와 TypeImportance, TypeCategory 필드가 함께 주어진 디렉터리 엔트리의 타입을 고유하게 식별함.

TypeImportance와 TypeCategory 필드가 모두 값 0을 포함하는 경우를 제외하고는 이 필드의 모든 가능한 값이 유효함; 그 경우 이 필드의 값 0은 무효임.

##### TypeImportance 필드
주어진 디렉터리 엔트리의 중요도를 설명함.

유효한 값:
- **0**: 주어진 디렉터리 엔트리가 중요함(critical)
- **1**: 주어진 디렉터리 엔트리가 무해함(benign)

##### TypeCategory 필드
주어진 디렉터리 엔트리의 카테고리를 설명함.

유효한 값:
- **0**: 주어진 디렉터리 엔트리가 primary임
- **1**: 주어진 디렉터리 엔트리가 secondary임

##### InUse 필드
주어진 디렉터리 엔트리가 사용 중인지 여부를 설명함.

유효한 값:
- **0**: 주어진 디렉터리 엔트리가 사용 중이 아님
- **1**: 주어진 디렉터리 엔트리가 사용 중임

#### FirstCluster 필드
주어진 디렉터리 엔트리와 연관된 Cluster Heap의 할당에서 첫 번째 클러스터의 인덱스를 포함함.

유효한 값 범위:
- **정확히 0**: 클러스터 할당이 존재하지 않음을 의미
- **2부터 ClusterCount + 1까지**: 유효한 클러스터 인덱스 범위

이 템플릿에서 파생된 구조들은 클러스터 할당이 파생 구조와 호환되지 않는 경우 FirstCluster와 DataLength 필드를 모두 재정의할 수 있음.

#### DataLength 필드
연관된 클러스터 할당이 포함하는 데이터의 크기를 바이트 단위로 설명함.

유효한 값 범위:
- **최소 0**: FirstCluster 필드가 값 0을 포함하면 이 필드의 유일한 유효한 값은 0임
- **최대 ClusterCount * 2^SectorsPerClusterShift * 2^BytesPerSectorShift**

이 템플릿에서 파생된 구조들은 파생 구조에 클러스터 할당이 불가능한 경우 FirstCluster와 DataLength 필드를 모두 재정의할 수 있음.

### Generic Primary DirectoryEntry Template
디렉터리 엔트리 세트의 첫 번째 디렉터리 엔트리는 primary 디렉터리 엔트리여야 함. 디렉터리 엔트리 세트의 모든 후속 디렉터리 엔트리들(있는 경우)은 secondary 디렉터리 엔트리여야 함.

Generic Primary DirectoryEntry template을 해석하는 능력은 필수임.

모든 primary 디렉터리 엔트리 구조는 Generic DirectoryEntry template에서 파생된 Generic Primary DirectoryEntry template에서 파생됨.

**Generic Primary DirectoryEntry Template**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|
| EntryType | 0 | 1 |
| SecondaryCount | 1 | 1 |
| SetChecksum | 2 | 2 |
| GeneralPrimaryFlags | 4 | 2 |
| CustomDefined | 6 | 14 |
| FirstCluster | 20 | 4 |
| DataLength | 24 | 8 |

#### EntryType 필드
Generic DirectoryEntry template에서 제공된 정의를 따름.[entry](#entrytype-필드)

##### Critical Primary Directory Entries
Critical primary 디렉터리 엔트리들은 exFAT 볼륨의 적절한 관리에 중요한 정보를 포함함. 루트 디렉터리만이 critical primary 디렉터리 엔트리들을 포함함(File 디렉터리 엔트리들은 예외).

Critical primary 디렉터리 엔트리들의 정의는 주요 exFAT 개정 번호와 연관됨. 구현체들은 모든 critical primary 디렉터리 엔트리들을 지원해야 하며 이 사양이 정의하는 critical primary 디렉터리 엔트리 구조만을 기록해야 함.

##### Benign Primary Directory Entries
Benign primary 디렉터리 엔트리들은 exFAT 볼륨 관리에 유용할 수 있는 추가 정보를 포함함. 어떤 디렉터리든 benign primary 디렉터리 엔트리들을 포함할 수 있음.

Benign primary 디렉터리 엔트리들의 정의는 부 exFAT 개정 번호와 연관됨. 이 사양이나 후속 사양이 정의하는 benign primary 디렉터리 엔트리에 대한 지원은 선택사항임.

##### TypeCategory 필드
주어진 디렉터리 엔트리의 카테고리를 설명함.

유효한 값:
- **0**: 주어진 디렉터리 엔트리가 primary임
- **1**: 주어진 디렉터리 엔트리가 secondary임

##### InUse 필드
주어진 디렉터리 엔트리가 사용 중인지 여부를 설명함.

유효한 값:
- **0**: 주어진 디렉터리 엔트리가 사용 중이 아님
- **1**: 주어진 디렉터리 엔트리가 사용 중임

#### SecondaryCount 필드
주어진 primary 디렉터리 엔트리 바로 다음에 오는 secondary 디렉터리 엔트리의 수를 설명함. 이러한 secondary 디렉터리 엔트리들이 주어진 primary 디렉터리 엔트리와 함께 디렉터리 엔트리 세트를 구성함.

유효한 값 범위:
- **최소 0**: 이 primary 디렉터리 엔트리가 디렉터리 엔트리 세트의 유일한 엔트리임을 의미
- **최대 255**: 다음 255개의 디렉터리 엔트리와 이 primary 디렉터리 엔트리가 디렉터리 엔트리 세트를 구성함을 의미

이 템플릿에서 파생된 Critical primary 디렉터리 엔트리 구조들은 SecondaryCount와 SetChecksum 필드를 모두 재정의할 수 있음.

#### SetChecksum 필드
주어진 디렉터리 엔트리 세트의 모든 디렉터리 엔트리들의 체크섬을 포함함. 하지만 체크섬은 이 필드를 제외함. 구현체들은 주어진 디렉터리 엔트리 세트의 다른 디렉터리 엔트리를 사용하기 전에 이 필드의 내용이 유효한지 확인해야 함.

```
UInt16 EntrySetChecksum
(
    UCHAR * Entries,       // points to an in-memory copy of the directory entry set
    UCHAR   SecondaryCount
)
{
    UInt16 NumberOfBytes = ((UInt16)SecondaryCount + 1) * 32;
    UInt16 Checksum = 0;
    UInt16 Index;

    for (Index = 0; Index < NumberOfBytes; Index++)
    {
        if ((Index == 2) || (Index == 3))
        {
            continue;
        }
        Checksum = ((Checksum&1) ? 0x8000 : 0) + (Checksum>>1) +  (UInt16)Entries[Index];
    }
    return Checksum;
}
```

#### GeneralPrimaryFlags 필드
플래그들을 포함함.

**Generic GeneralPrimaryFlags Field Structure**

| 필드 이름 | 오프셋(비트) | 크기(비트) |
|-------|---------------|-------------|
| AllocationPossible | 0 | 1 |
| NoFatChain | 1 | 1 |
| CustomDefined | 2 | 14 |

##### AllocationPossible 필드
주어진 디렉터리 엔트리에 대해 Cluster Heap의 할당이 가능한지 여부를 설명함.

유효한 값:
- **0**: 연관된 클러스터 할당이 불가능하며 FirstCluster와 DataLength 필드는 실제로 정의되지 않음
- **1**: 연관된 클러스터 할당이 가능하며 FirstCluster와 DataLength 필드는 정의된 대로임

##### NoFatChain 필드
활성 FAT가 주어진 할당의 클러스터 체인을 설명하는지 여부를 나타냄.

유효한 값:
- **0**: 할당의 클러스터 체인에 대한 해당 FAT 엔트리들이 유효하며 구현체들은 이를 해석해야 함
- **1**: 연관된 할당이 하나의 연속된 클러스터 시리즈임; 클러스터들에 대한 해당 FAT 엔트리들은 무효하며 구현체들은 이를 해석하지 말아야 함

#### FirstCluster 필드
FirstCluster 필드는 Generic DirectoryEntry template에서 제공된 정의를 따라야 함.

NoFatChain 비트가 1이면 FirstCluster는 클러스터 힙의 유효한 클러스터를 가리켜야 함.

이 템플릿에서 파생된 Critical primary 디렉터리 엔트리 구조들은 FirstCluster와 DataLength 필드를 재정의할 수 있음. 이 템플릿에서 파생된 다른 구조들은 AllocationPossible 필드가 값 0을 포함하는 경우에만 FirstCluster와 DataLength 필드를 재정의할 수 있음.

#### DataLength 필드
DataLength 필드는 Generic DirectoryEntry template에서 제공된 정의를 따라야 함.

NoFatChain 비트가 1이면 DataLength는 0이 아니어야 함. FirstCluster 필드가 0이면 DataLength도 0이어야 함.

이 템플릿에서 파생된 Critical primary 디렉터리 엔트리 구조들은 FirstCluster와 DataLength 필드를 재정의할 수 있음. 이 템플릿에서 파생된 다른 구조들은 AllocationPossible 필드가 값 0을 포함하는 경우에만 FirstCluster와 DataLength 필드를 재정의할 수 있음.

### Generic Secondary DirectoryEntry Template
Secondary 디렉터리 엔트리들의 중심 목적은 디렉터리 엔트리 세트에 대한 추가 정보를 제공하는 것임. Generic Secondary DirectoryEntry template을 해석하는 능력은 필수임.

Critical과 benign secondary 디렉터리 엔트리들 모두의 정의는 minor exFAT 개정 번호와 연관됨. 이 사양이나 후속 사양이 정의하는 critical 또는 benign secondary 디렉터리 엔트리에 대한 지원은 선택사항임.

모든 secondary 디렉터리 엔트리 구조는 Generic DirectoryEntry template에서 파생된 Generic Secondary DirectoryEntry template에서 파생됨.

**Generic Secondary DirectoryEntry Template**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| EntryType | 0 | 1 |
| GeneralSecondaryFlags | 1 | 1 |
| CustomDefined | 2 | 18 |
| FirstCluster | 20 | 4 |
| DataLength | 24 | 8 |

#### Critical Secondary Directory Entries
Critical secondary 디렉터리 엔트리들은 포함하고 있는 디렉터리 엔트리 세트의 적절한 관리에 중요한 정보를 포함함. 특정 critical secondary 디렉터리 엔트리에 대한 지원은 선택사항이지만, 인식되지 않는 critical 디렉터리 엔트리는 전체 디렉터리 엔트리 세트를 인식되지 않는 것으로 만듦.

#### Benign Secondary Directory Entries
Benign secondary 디렉터리 엔트리들은 포함하고 있는 디렉터리 엔트리 세트의 관리에 유용할 수 있는 추가 정보를 포함함. 특정 benign secondary 디렉터리 엔트리에 대한 지원은 선택사항임. 인식되지 않는 benign secondary 디렉터리 엔트리들은 전체 디렉터리 엔트리 세트를 인식되지 않는 것으로 만들지 않음.

구현들은 인식하지 못하는 benign secondary 엔트리를 무시할 수 있음

##### TypeCategory 필드
Generic DirectoryEntry template에서 제공된 정의를 따름.

이 템플릿에서 이 필드의 유효한 값은 1임.

##### InUse 필드
Generic DirectoryEntry template에서 제공된 정의를 따름.

#### GeneralSecondaryFlags 필드
플래그들을 포함함.

**Generic GeneralSecondaryFlags Field Structure**

| 필드 이름 | 오프셋(비트) | 크기(비트) | 설명 |
|-------|---------------|-------------|------|
| AllocationPossible | 0 | 1 | 필수 필드, 내용은 섹션 6.4.2.1에서 정의 |
| NoFatChain | 1 | 1 | 필수 필드, 내용은 섹션 6.4.2.2에서 정의 |
| CustomDefined | 2 | 6 | 필수 필드, 이 템플릿에서 파생된 구조가 이 필드를 정의할 수 있음 |

##### AllocationPossible 필드
Generic Primary DirectoryEntry template의 동일한 이름 필드와 같은 정의를 가짐.

##### NoFatChain 필드
Generic Primary DirectoryEntry template의 동일한 이름 필드와 같은 정의를 가짐.

#### FirstCluster 필드
Generic DirectoryEntry template에서 제공된 정의를 따름.

NoFatChain 비트가 1이면 FirstCluster는 클러스터 힙의 유효한 클러스터를 가리켜야 함.

#### DataLength 필드
Generic DirectoryEntry template에서 제공된 정의를 따름.

NoFatChain 비트가 1이면 DataLength는 0이 아니어야 함. FirstCluster 필드가 0이면 DataLength도 0이어야 함.

## Directory Entry Definitions
exFAT 파일시스템 버전 1.00은 다음과 같은 디렉터리 엔트리들을 정의함:

**Critical primary**
- Allocation Bitmap 
- Up-case Table 
- Volume Label 
- File 

**Benign primary**
- Volume GUID 
- TexFAT Padding 

**Critical secondary**
- Stream Extension 
- File Name 

**Benign secondary**
- Vendor Extension 
- Vendor Allocation

### Allocation Bitmap Directory Entry
exFAT 파일시스템에서는 FAT가 클러스터의 할당 상태를 설명하지 않고 Allocation Bitmap이 담당함. Allocation Bitmap은 Cluster Heap에 존재하며 루트 디렉터리에 해당하는 critical primary 디렉터리 엔트리를 가짐.

NumberOfFats 필드가 루트 디렉터리의 유효한 Allocation Bitmap 디렉터리 엔트리 수를 결정함. NumberOfFats 필드가 값 1을 포함하면 유효한 Allocation Bitmap 디렉터리 엔트리 수는 1개뿐임. 또한 하나의 Allocation Bitmap 디렉터리 엔트리는 First Allocation Bitmap을 설명하는 경우에만 유효함. NumberOfFats 필드가 값 2를 포함하면 유효한 Allocation Bitmap 디렉터리 엔트리 수는 2개뿐임. 또한 두 Allocation Bitmap 디렉터리 엔트리는 하나는 First Allocation Bitmap을, 다른 하나는 Second Allocation Bitmap을 설명하는 경우에만 유효함.

**Allocation Bitmap DirectoryEntry Structure**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| EntryType | 0 | 1 |
| BitmapFlags | 1 | 1 |
| Reserved | 2 | 18 |
| FirstCluster | 20 | 4 |
| DataLength | 24 | 8 |

#### EntryType 필드
Generic Primary DirectoryEntry template에서 제공된 정의를 따름.

##### TypeCode 필드
Allocation Bitmap 디렉터리 엔트리의 경우 이 필드의 유효한 값은 1임.

##### TypeImportance 필드
Allocation Bitmap 디렉터리 엔트리의 경우 이 필드의 유효한 값은 0임.

#### BitmapFlags 필드
플래그들을 포함함.

**BitmapFlags Field Structure**

| 필드 이름 | 오프셋(비트) | 크기(비트) | 설명 |
|-------|---------------|-------------|------|
| BitmapIdentifier | 0 | 1 | 필수 필드, 내용은 섹션 7.1.2.1에서 정의 |
| Reserved | 1 | 7 | 필수 필드, 내용은 예약됨 |

##### BitmapIdentifier 필드
주어진 디렉터리 엔트리가 설명하는 Allocation Bitmap을 나타냄. 구현체들은 First Allocation Bitmap을 First FAT와 함께 사용해야 하며 Second Allocation Bitmap을 Second FAT와 함께 사용해야 함. ActiveFat 필드가 어떤 FAT와 Allocation Bitmap이 활성화되어 있는지 설명함.

유효한 값:
- **0**: 주어진 디렉터리 엔트리가 First Allocation Bitmap을 설명함을 의미
- **1**: 주어진 디렉터리 엔트리가 Second Allocation Bitmap을 설명함을 의미하며 NumberOfFats가 값 2를 포함할 때만 가능함

#### FirstCluster 필드
Generic Primary DirectoryEntry template에서 제공된 정의를 따름.

이 필드는 FAT가 설명하는 대로 Allocation Bitmap을 호스트하는 클러스터 체인의 첫 번째 클러스터의 인덱스를 포함함.

#### DataLength 필드
Generic Primary DirectoryEntry template에서 제공된 정의를 따름.

#### Allocation Bitmap
Allocation Bitmap은 Cluster Heap의 클러스터들의 할당 상태를 기록함. Allocation Bitmap의 각 비트는 해당 클러스터가 할당 가능한지 여부를 나타냄.

Allocation Bitmap은 클러스터를 가장 낮은 인덱스부터 가장 높은 인덱스 순서로 나타냄. 역사적 이유로 첫 번째 클러스터는 인덱스 2를 가짐.

**참고:** 비트맵의 첫 번째 비트는 첫 번째 바이트의 최하위 비트임.

**Allocation Bitmap Structure**

| 필드 이름 | 오프셋(비트) | 크기(비트) |
|-------|---------------|-------------|
| BitmapEntry[2] | 0 | 1 |
| . | . | . |
| . | . | . |
| . | . | . |
| BitmapEntry[ClusterCount+1] | ClusterCount - 1 | 1 |
| Reserved | ClusterCount | (DataLength * 8) – ClusterCount |

##### BitmapEntry[2] ... BitmapEntry[ClusterCount+1] 필드들
이 배열의 각 BitmapEntry 필드는 Cluster Heap의 클러스터를 나타냄. BitmapEntry[2]는 Cluster Heap의 첫 번째 클러스터를 나타내고 BitmapEntry[ClusterCount+1]은 Cluster Heap의 마지막 클러스터를 나타냄.

유효한 값:
- **0**: 해당 클러스터가 할당 가능함을 설명
- **1**: 해당 클러스터가 할당 불가능함을 설명 (클러스터 할당이 이미 해당 클러스터를 소비했거나 활성 FAT가 해당 클러스터를 배드로 설명할 수 있음)

### Up-case Table Directory Entry
Up-case Table은 소문자에서 대문자로의 변환을 정의함. 이는 File Name 디렉터리 엔트리가 Unicode 문자를 사용하고 exFAT 파일시스템이 대소문자를 구분하지 않으면서 대소문자를 보존하기 때문에 중요함. Up-case Table은 Cluster Heap에 존재하며 루트 디렉터리에 해당하는 critical primary 디렉터리 엔트리를 가짐. 유효한 Up-case Table 디렉터리 엔트리 수는 1개임.

Up-case Table과 파일 이름 간의 관계로 인해 구현체들은 포맷 작업의 결과를 제외하고는 Up-case Table을 수정해서는 안 됨.

**Up-case Table DirectoryEntry Structure**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) | 설명 |
|-------|---------------|-------------|------|
| EntryType | 0 | 1 | 필수 필드, 내용은 섹션 7.2.1에서 정의 |
| Reserved1 | 1 | 3 | 필수 필드, 내용은 예약됨 |
| TableChecksum | 4 | 4 | 필수 필드, 내용은 섹션 7.2.2에서 정의 |
| Reserved2 | 8 | 12 | 필수 필드, 내용은 예약됨 |
| FirstCluster | 20 | 4 | 필수 필드, 내용은 섹션 7.2.3에서 정의 |
| DataLength | 24 | 8 | 필수 필드, 내용은 섹션 7.2.4에서 정의 |

#### EntryType 필드
Up-case Table 디렉터리 엔트리의 경우 TypeCode 필드의 유효한 값은 2이고, TypeImportance 필드의 유효한 값은 0임.

#### TableChecksum 필드
FirstCluster와 DataLength 필드가 설명하는 Up-case Table의 체크섬을 포함함. 구현체들은 Up-case Table을 사용하기 전에 이 필드의 내용이 유효한지 확인해야 함.

#### FirstCluster 필드
이 필드는 FAT가 설명하는 대로 Up-case Table을 호스트하는 클러스터 체인의 첫 번째 클러스터의 인덱스를 포함함.

#### Up-case Table
Up-case table은 일련의 Unicode 문자 매핑임. 문자 매핑은 2바이트 필드로 구성되며, up-case table에서 필드의 인덱스는 대문자화될 Unicode 문자를 나타내고, 2바이트 필드는 대문자화된 Unicode 문자를 나타냄.

처음 128개의 Unicode 문자는 필수 매핑을 가짐. 처음 128개의 Unicode 문자 중 어떤 것에 대해서라도 다른 문자 매핑을 가진 up-case table은 무효함.

필수 매핑 범위의 문자만 지원하는 구현체들은 up-case table의 나머지 매핑을 무시할 수 있음. 이러한 구현체들은 파일을 생성하거나 이름을 바꿀 때 필수 매핑 범위의 문자만 사용해야 함.

볼륨 포맷 시 구현체들은 압축된 형식으로 up-case table을 생성할 수 있음. 구현체들은 일련의 항등 매핑을 값 FFFFh와 항등 매핑의 수로 나타내어 up-case table을 압축함.

### Volume Label Directory Entry
Volume Label은 최종 사용자가 저장 볼륨을 구별할 수 있게 하는 Unicode 문자열임. exFAT 파일시스템에서 Volume Label은 루트 디렉터리의 critical primary 디렉터리 엔트리로 존재함. 유효한 Volume Label 디렉터리 엔트리 수는 0에서 1까지임.

**Volume Label DirectoryEntry Structure**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) | 설명 |
|-------|---------------|-------------|------|
| EntryType | 0 | 1 | 필수 필드, 내용은 섹션 7.3.1에서 정의 |
| CharacterCount | 1 | 1 | 필수 필드, 내용은 섹션 7.3.2에서 정의 |
| VolumeLabel | 2 | 22 | 필수 필드, 내용은 섹션 7.3.3에서 정의 |
| Reserved | 24 | 8 | 필수 필드, 내용은 예약됨 |

#### EntryType 필드
Volume Label 디렉터리 엔트리의 경우 TypeCode 필드의 유효한 값은 3이고, TypeImportance 필드의 유효한 값은 0임.

#### CharacterCount 필드
VolumeLabel 필드가 포함하는 Unicode 문자열의 길이를 포함함.

유효한 값 범위:
- **최소 0**: Unicode 문자열이 0자 길이임을 의미 (볼륨 레이블 없음과 동등)
- **최대 11**: Unicode 문자열이 11자 길이임을 의미

#### VolumeLabel 필드
볼륨의 사용자 친화적 이름인 Unicode 문자열을 포함함. VolumeLabel 필드는 File Name 디렉터리 엔트리의 FileName 필드와 같은 무효 문자 집합을 가짐.

### File Directory Entry
File 디렉터리 엔트리들은 파일과 디렉터리를 설명함. 이들은 critical primary 디렉터리 엔트리이며 어떤 디렉터리든 0개 이상의 File 디렉터리 엔트리를 포함할 수 있음. File 디렉터리 엔트리가 유효하려면 정확히 하나의 Stream Extension 디렉터리 엔트리와 최소 하나의 File Name 디렉터리 엔트리가 File 디렉터리 엔트리 바로 다음에 와야 함.

**File DirectoryEntry Structure**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| EntryType | 0 | 1 |
| SecondaryCount | 1 | 1 |
| SetChecksum | 2 | 2 |
| FileAttributes | 4 | 2 |
| Reserved1 | 6 | 2 |
| CreateTimestamp | 8 | 4 |
| LastModifiedTimestamp | 12 | 4 |
| LastAccessedTimestamp | 16 | 4 |
| Create10msIncrement | 20 | 1 |
| LastModified10msIncrement | 21 | 1 |
| CreateUtcOffset | 22 | 1 |
| LastModifiedUtcOffset | 23 | 1 |
| LastAccessedUtcOffset | 24 | 1 |
| Reserved2 | 25 | 7 |

#### EntryType 필드
File 디렉터리 엔트리의 경우 TypeCode 필드의 유효한 값은 5이고, TypeImportance 필드의 유효한 값은 0임.

#### FileAttributes 필드
플래그들을 포함함.

**FileAttributes Field Structure**

| 필드 이름 | 오프셋(비트) | 크기(비트) |
|-------|---------------|-------------|
| ReadOnly | 0 | 1 |
| Hidden | 1 | 1 |
| System | 2 | 1 |
| Reserved1 | 3 | 1 |
| Directory | 4 | 1 |
| Archive | 5 | 1 |
| Reserved2 | 6 | 10 |

#### Timestamp 필드들
Timestamp 필드들은 2초 해상도까지 로컬 날짜와 시간을 모두 설명함.

**Timestamp Field Structure**

| 필드 이름 | 오프셋(비트) | 크기(비트) |
|-------|---------------|-------------|
| DoubleSeconds | 0 | 5 |
| Hour | 11 | 5 |
| Day | 16 | 5 |
| Month | 21 | 4 |
| Year | 25 | 7 |

#### 10msIncrement 필드들
10msIncrement 필드들은 해당 Timestamp 필드에 10밀리초 배수로 추가 시간 해상도를 제공함.

유효한 값 범위: 0 (0밀리초)에서 199 (1990밀리초)까지

#### UtcOffset 필드들
UtcOffset 필드들은 해당 Timestamp와 10msIncrement 필드가 설명하는 로컬 날짜와 시간으로부터 UTC까지의 오프셋을 설명함.

**UtcOffset Field Structure**

| 필드 이름 | 오프셋(비트) | 크기(비트) |
|-------|---------------|-------------|
| OffsetFromUtc | 0 | 7 |
| OffsetValid | 7 | 1 |

### Volume GUID Directory Entry
Volume GUID 디렉터리 엔트리는 구현체들이 볼륨을 고유하고 프로그래밍적으로 구별할 수 있게 하는 GUID를 포함함. Volume GUID는 루트 디렉터리의 benign primary 디렉터리 엔트리로 존재함. 유효한 Volume GUID 디렉터리 엔트리 수는 0에서 1까지임.

### Stream Extension Directory Entry
Stream Extension 디렉터리 엔트리는 File 디렉터리 엔트리 세트의 critical secondary 디렉터리 엔트리임. File 디렉터리 엔트리 세트의 유효한 Stream Extension 디렉터리 엔트리 수는 1개임. 또한 이 디렉터리 엔트리는 File 디렉터리 엔트리 바로 다음에 오는 경우에만 유효함.

**Stream Extension DirectoryEntry Structure**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) |
|-------|---------------|-------------|
| EntryType | 0 | 1 |
| GeneralSecondaryFlags | 1 | 1 |
| Reserved1 | 2 | 1 |
| NameLength | 3 | 1 |
| NameHash | 4 | 2 |
| Reserved2 | 6 | 2 |
| ValidDataLength | 8 | 8 |
| Reserved3 | 16 | 4 |
| FirstCluster | 20 | 4 |
| DataLength | 24 | 8 |

#### NameLength 필드
후속 File Name 디렉터리 엔트리들이 집합적으로 포함하는 Unicode 문자열의 길이를 포함함.

유효한 값 범위: 1 (가장 짧은 파일 이름)에서 255 (가장 긴 파일 이름)까지

#### NameHash 필드
대문자화된 파일 이름의 2바이트 해시를 포함함. 이는 구현체들이 이름으로 파일을 검색할 때 빠른 비교를 수행할 수 있게 함.

#### ValidDataLength 필드
데이터 스트림에 사용자 데이터가 얼마나 멀리 쓰여졌는지 설명함. 구현체들은 데이터 스트림에 더 멀리 데이터를 쓸 때 이 필드를 업데이트해야 함.

#### FirstCluster 필드
사용자 데이터를 호스트하는 데이터 스트림의 첫 번째 클러스터의 인덱스를 포함함.

### File Name Directory Entry
File Name 디렉터리 엔트리들은 File 디렉터리 엔트리 세트의 critical secondary 디렉터리 엔트리임. File 디렉터리 엔트리 세트의 유효한 File Name 디렉터리 엔트리 수는 NameLength / 15를 가장 가까운 정수로 반올림한 값임. 또한 File Name 디렉터리 엔트리들은 Stream Extension 디렉터리 엔트리 바로 다음에 연속적인 시리즈로 오는 경우에만 유효함.

**File Name DirectoryEntry Structure**

| 필드 이름 | 오프셋(바이트) | 크기(바이트) | 설명 |
|-------|---------------|-------------|------|
| EntryType | 0 | 1 | 필수 필드, 내용은 섹션 7.7.1에서 정의 |
| GeneralSecondaryFlags | 1 | 1 | 필수 필드, 내용은 섹션 7.7.2에서 정의 |
| FileName | 2 | 30 | 필수 필드, 내용은 섹션 7.7.3에서 정의 |

#### FileName 필드
파일 이름의 일부인 Unicode 문자열을 포함함. File 디렉터리 엔트리 세트에 File Name 디렉터리 엔트리들이 존재하는 순서대로 FileName 필드들이 연결되어 File 디렉터리 엔트리 세트의 파일 이름을 형성함.

연결된 파일 이름은 다른 FAT 기반 파일시스템과 같은 불법 문자 집합을 가짐. 사용되지 않는 FileName 필드의 문자들은 값 0000h로 설정해야 함.

**무효한 FileName 문자들**
- 제어 코드 (0000h~001Fh)
- 따옴표 (0022h)
- 별표 (002Ah)
- 슬래시 (002Fh)
- 콜론 (003Ah)
- 부등호 (003Ch, 003Eh)
- 물음표 (003Fh)
- 백슬래시 (005Ch)
- 수직 바 (007Ch)

파일 이름 "."과 ".."는 각각 "이 디렉터리"와 "포함하는 디렉터리"의 특별한 의미를 가짐. 구현체들은 이러한 예약된 파일 이름 중 어떤 것도 FileName 필드에 기록해서는 안 됨.

### Vendor Extension Directory Entry
Vendor Extension 디렉터리 엔트리는 File 디렉터리 엔트리 세트의 benign secondary 디렉터리 엔트리임. File 디렉터리 엔트리 세트는 secondary 디렉터리 엔트리의 한계에서 다른 secondary 디렉터리 엔트리의 수를 뺀 만큼까지 임의의 수의 Vendor Extension 디렉터리 엔트리를 포함할 수 있음.

Vendor Extension 디렉터리 엔트리들은 벤더들이 VendorGuid 필드를 통해 개별 File 디렉터리 엔트리 세트에 고유한 벤더별 디렉터리 엔트리를 가질 수 있게 함.

### Vendor Allocation Directory Entry
Vendor Allocation 디렉터리 엔트리는 File 디렉터리 엔트리 세트의 benign secondary 디렉터리 엔트리임. Vendor Extension과 유사하지만 연관된 클러스터들을 가질 수 있음.

### TexFAT Padding Directory Entry
이 사양인 exFAT 버전 1.00 파일시스템 기본 사양은 TexFAT Padding 디렉터리 엔트리를 정의하지 않음. 그러나 그 type code는 1이고 type importance는 1임. 이 사양의 구현체들은 TexFAT Padding 디렉터리 엔트리를 다른 인식되지 않는 benign primary 디렉터리 엔트리와 같이 처리해야 하며, 구현체들은 TexFAT Padding 디렉터리 엔트리를 이동시켜서는 안 됨.

# Linux 커널 코드 분석
```
osvs@BoB:~/OSVS/OSVS_EventHorizon/lkl/fs/exfat$ tree
.
├── balloc.c
├── balloc.o
├── built-in.a
├── cache.c
├── cache.o
├── dir.c
├── dir.o
├── exfat_fs.h
├── exfat_raw.h
├── fatent.c
├── fatent.o
├── file.c
├── file.o
├── inode.c
├── inode.o
├── Kconfig
├── Makefile
├── misc.c
├── misc.o
├── namei.c
├── namei.o
├── nls.c
├── nls.o
├── super.c
└── super.o

1 directory, 25 files
osvs@BoB:~/OSVS/OSVS_EventHorizon/lkl/fs/exfat$ 
```
## 아이노드
리눅스에서는 **VFS(Virtual File System)** 추상화 계층을 사용한다. VFS는 여러 파일 시스템을 동일한 방식으로 다룰 수 있게 해주는 일종의 **표준 인터페이스** 라고 볼 수도 있다. 클로드가 그린 구조는 다음과 같다. 
```
● 🏗️ Linux VFS 계층 구조
  VFS 계층 구조

  👤 사용자 프로그램
      │
      ├─ open(), read(), write()...
      │
  📋 VFS (Virtual File System)
      │ ┌─────────────────────────────┐
      │ │ 공통 아이노드 (struct inode) │ <- 표준 인터페이스
      │ └─────────────────────────────┘
      │
  🔗 파일시스템별 구현체
      │
      ├─ ext4: ext4_inode_info        <- 확장
      ├─ xfs:  xfs_inode              <- 확장
      ├─ btrfs: btrfs_inode           <- 확장
      └─ exFAT: exfat_inode_info      <- 확장
           │
      💾 실제 디스크 (exFAT 포맷)
```

`include/linux/fs.h`에 커널에서 일반적으로 사용하는 **inode** 구조체를 확인할 수 있다. inode는 파일의 메타데이터를 저장하는 데이터 구조로 파일 하나당 하나씩 존재하는 credential의 느낌 같다. 파일 이름과 실제 데이터는 저장하지 않는다. 
```
/*
 * Keep mostly read-only and often accessed (especially for
 * the RCU path lookup and 'stat' data) fields at the beginning
 * of the 'struct inode'
 */
struct inode {
	umode_t			i_mode;
	unsigned short		i_opflags;
	kuid_t			i_uid;
	kgid_t			i_gid;
	unsigned int		i_flags;
...

```

이런식으로 구조체가 선언이 되어 있다. 주요 필드만 살펴보자면
- umode_t i_mode: 파일 타입과 접근 권한
- kuid_t i_uid: User ID
- kgid_t i_gid: Group ID
- loff_t i_sze: 파일 크기를 바이트 단위로
- unsigned long i_ino; 필드에 이 inode의 **고유 번호** 를 저장함

그리고 `fs/exfat/exfat_fs.h`에는 위의 inode 구조체를 확장한 구조체가 선언되어 있다. inode가 공통적으로 담지 않는 exFAT의 고유한 특징을 위해 필요한 필드를 포함하고 있다. 

```
/*
 * EXFAT file system inode in-memory data
 */
struct exfat_inode_info {
	/* the cluster where file dentry is located */
	struct exfat_chain dir;
	/* the index of file dentry in ->dir */
	int entry;
	unsigned int type;
	unsigned short attr;
	unsigned int start_clu;
	unsigned char flags;
	/*
	 * the copy of low 32bit of i_version to check
	 * the validation of hint_stat.
	 */
	unsigned int version;

	/* hint for cluster last accessed */
	struct exfat_hint hint_bmap;
	/* hint for entry index we try to lookup next time */
	struct exfat_hint hint_stat;
	/* hint for first empty entry */
	struct exfat_hint_femp hint_femp;

	spinlock_t cache_lru_lock;
	struct list_head cache_lru;
	int nr_caches;
	/* for avoiding the race between alloc and free */
	unsigned int cache_valid_id;

	/* on-disk position of directory entry or 0 */
	loff_t i_pos;
	loff_t valid_size;
	/* hash by i_location */
	struct hlist_node i_hash_fat;
	/* protect bmap against truncate */
	struct rw_semaphore truncate_lock;
	struct inode vfs_inode;
	/* File creation time */
	struct timespec64 i_crtime;
};
```
- **struct inode vfs_inode**: VFS가 사용하는 표준, 범용 inode로 모든 파일 시스템이 공통적으로 갖는 정보 포함
- **struct exfat_inode_info**: exFAT 만의 고유한 특징을 담고 있음

## 헤더파일 분석
### exfat_raw.h
**온디스크 포맷**, 즉 실제 디스크에 저장되는 데이터 구조를 정의하고 있다. `__packed` 속성으로 메모리 정렬 없이 정확한 바이트 레이아웃을 보장한다. 정적 상수들은 명세에 정의된 값들이다. 

```
#define BOOT_SIGNATURE		0xAA55
#define EXBOOT_SIGNATURE	0xAA550000
#define STR_EXFAT		"EXFAT   "	/* size should be 8 */

#define EXFAT_MAX_FILE_LEN	255

#define VOLUME_DIRTY		0x0002
#define MEDIA_FAILURE		0x0004

#define EXFAT_EOF_CLUSTER	0xFFFFFFFFu
#define EXFAT_BAD_CLUSTER	0xFFFFFFF7u
#define EXFAT_FREE_CLUSTER	0
/* Cluster 0, 1 are reserved, the firs
```
- BOOT_SIGNATURE: 부트섹터 시그니처(마지막 2바이트)
- EXBOOT_SIGNATURE: 확장 부트섹터 시그니처
- STR_EXFAT: 파일시스템 이름 (8바이트 고정)
- EXFAT_EOF_CLUSTER: **0xFFFFFFFFu** 클러스터 체인 끝
- EXFAT_BAD_CLUSTER: **0xFFFFFFF7u** 손상된 클러스터
- EXFAT_FREE_CLUSTER: 0, 빈 클러스터 

```
/* Cluster 0, 1 are reserved, the first cluster is 2 in the cluster heap. */
#define EXFAT_RESERVED_CLUSTERS	2
#define EXFAT_FIRST_CLUSTER	2
#define EXFAT_DATA_CLUSTER_COUNT(sbi)	\
	((sbi)->num_clusters - EXFAT_RESERVED_CLUSTERS)
```
- EXFAT_RESERVED_CLUSTERS, EXFAT_FIRST_CLUSTER => 실제로 쓰는 클러스터는 인덱스 2부터..
- EXFAT_DATA_CLUSTER_COUNT: 실제 사용 가능한 데이터 클러스터 개수 계산(전체 클러스터 - 2)

```
/* AllocationPossible and NoFatChain field in GeneralSecondaryFlags Field */
#define ALLOC_POSSIBLE		0x01
#define ALLOC_FAT_CHAIN		0x01
#define ALLOC_NO_FAT_CHAIN	0x03
```
**스트림 엔트리** 의 flags 필드
| 필드 이름 | 오프셋(비트) | 크기(비트) |
|-------|---------------|-------------|
| AllocationPossible | 0 | 1 |
| NoFatChain | 1 | 1 |
| CustomDefined | 2 | 14 |

- 0x1: 0001, FAT 엔트리 유효 + 반드시 해석 필요 + FAT 테이블 순회해서 다음 클러스터 찾기.
- 0x3: 0011, FAT 엔트리 무효화 + 클러스터 연결해서 순차적 할당. 

```
static void exfat_init_stream_entry(struct exfat_dentry *ep,
		unsigned int start_clu, unsigned long long size)
{
	memset(ep, 0, sizeof(*ep));
	exfat_set_entry_type(ep, TYPE_STREAM);
	if (size == 0)
		ep->dentry.stream.flags = ALLOC_FAT_CHAIN;
	else
		ep->dentry.stream.flags = ALLOC_NO_FAT_CHAIN;
	ep->dentry.stream.start_clu = cpu_to_le32(start_clu);
	ep->dentry.stream.valid_size = cpu_to_le64(size);
	ep->dentry.stream.size = cpu_to_le64(size);
}
```
위 코드는 `Stream Extension Directory Entry`를 초기화하는 함수이다. 엔트리 타입을 **0xC0(TYPE_STREAM)** 으로 설정해주고 파일 크기에 따라서 FAT 사용 여부를 결정하고 있다. 크기가 0이 아니라면 연속 할당을 기본적으로 사용하고 있다.

```
#define DENTRY_SIZE		32 /* directory entry size */
#define DENTRY_SIZE_BITS	5
```
- DENTRY_SIZE: 모든 디렉터리 엔트리는 정확히 32바이트 고정
- DENTRY_SIZE_BITS: 2^5 = 32. 빠른 계산을 위한 헬퍼라고 보면 될 듯

```
/* dentry types */
#define EXFAT_UNUSED		0x00	/* end of directory */
#define EXFAT_DELETE		(~0x80)
#define IS_EXFAT_DELETED(x)	((x) < 0x80) /* deleted file (0x01~0x7F) */
#define EXFAT_INVAL		0x80	/* invalid value */
#define EXFAT_BITMAP		0x81	/* allocation bitmap */
#define EXFAT_UPCASE		0x82	/* upcase table */
#define EXFAT_VOLUME		0x83	/* volume label */
#define EXFAT_FILE		0x85	/* file or dir */
#define EXFAT_GUID		0xA0
#define EXFAT_PADDING		0xA1
#define EXFAT_ACLTAB		0xA2
#define EXFAT_STREAM		0xC0	/* stream entry */
#define EXFAT_NAME		0xC1	/* file name entry */
#define EXFAT_ACL		0xC2	/* stream entry */
#define EXFAT_VENDOR_EXT	0xE0	/* vendor extension entry */
#define EXFAT_VENDOR_ALLOC	0xE1	/* vendor allocation entry */

#define IS_EXFAT_CRITICAL_PRI(x)	(x < 0xA0)
#define IS_EXFAT_BENIGN_PRI(x)		(x < 0xC0)
#define IS_EXFAT_CRITICAL_SEC(x)	(x < 0xE0)
```
exFAT 디렉터리 엔트리의 타입 분류 체계를 정의해주고 있다. **EntryType** 필드에 들어가는 값들이다. [1](#entrytype-필드). 처음 한바이트를 보고 바로 어떤 디렉터리 엔트리인지 구분할 수 있도록 해둔 상수들이다. 아래와 같이 분류할 수 있다.
- 0x81-0x9F: Critical Primary (필수 주 엔트리)
- 0xA0-0xBF: Benign Primary (선택적 주 엔트리)
- 0xC0-0xDF: Critical Secondary (필수 부 엔트리)
- 0xE0-0xFF: Benign Secondary (선택적 부 엔트리)

```
/* checksum types */
#define CS_DIR_ENTRY		0
#define CS_BOOT_SECTOR		1
#define CS_DEFAULT		2
```
체크섬 계산 알고리즘의 타입을 구분하는 부분이다. `디렉터리`, `부트 섹터`, `기본, 기타` 체크섬 계산 방식에 구분을 두기 위한 상수이다.

```
/* file attributes */
#define EXFAT_ATTR_READONLY	0x0001
#define EXFAT_ATTR_HIDDEN	0x0002
#define EXFAT_ATTR_SYSTEM	0x0004
#define EXFAT_ATTR_VOLUME	0x0008
#define EXFAT_ATTR_SUBDIR	0x0010
#define EXFAT_ATTR_ARCHIVE	0x0020

#define EXFAT_ATTR_RWMASK	(EXFAT_ATTR_HIDDEN | EXFAT_ATTR_SYSTEM | \
				 EXFAT_ATTR_VOLUME | EXFAT_ATTR_SUBDIR | \
				 EXFAT_ATTR_ARCHIVE)
```
`FILE` 엔트리의 **attr** 필드에서 사용됨. 파일 속성 플래그들을 정의하고 있다. **EXFAT_ATTR_RWMASK** 는 READONLY를 별도로 처리하려고 만든 마스크 같다. 

```
#define BOOTSEC_JUMP_BOOT_LEN		3
#define BOOTSEC_FS_NAME_LEN		8
#define BOOTSEC_OLDBPB_LEN		53

#define EXFAT_FILE_NAME_LEN		15

#define EXFAT_MIN_SECT_SIZE_BITS		9
#define EXFAT_MAX_SECT_SIZE_BITS		12
#define EXFAT_MAX_SECT_PER_CLUS_BITS(x)		(25 - (x)->sect_size_bits)
```
- BOOTSEC_JUMP_BOOT_LEN: 점프 명령어는 3바이트
- BOOTSEC_FS_NAME_LEN: `EXFAT  ` 문자열은 8바이트
- BOOTSEC_OLDBPB_LEN: 이전 FAT 호환 영역 53바이트
- EXFAT_FILE_NAME_LEN: **NAME 엔트리** 당 문자 수 15개(30바이트)
- EXFAT_MIN/MAX_SECT_SIZE_BITS: 섹터 크기 최소 512, 최대 4096
- EXFAT_MAX_SECT_PER_CLUS_BITS(x): 클러스터 당 섹터 수 제한

```
/* EXFAT: Main and Backup Boot Sector (512 bytes) */
struct boot_sector {
	__u8	jmp_boot[BOOTSEC_JUMP_BOOT_LEN];
	__u8	fs_name[BOOTSEC_FS_NAME_LEN];
	__u8	must_be_zero[BOOTSEC_OLDBPB_LEN];
	__le64	partition_offset;
	__le64	vol_length;
	__le32	fat_offset;
	__le32	fat_length;
	__le32	clu_offset;
	__le32	clu_count;
	__le32	root_cluster;
	__le32	vol_serial;
	__u8	fs_revision[2];
	__le16	vol_flags;
	__u8	sect_size_bits;
	__u8	sect_per_clus_bits;
	__u8	num_fats;
	__u8	drv_sel;
	__u8	percent_in_use;
	__u8	reserved[7];
	__u8	boot_code[390];
	__le16	signature;
} __packed;
```
**BOOT SECTOR** 레이아웃을 정의하고 있음. `__le64`, `__le32`, `__le16`은 리틀 엔디안 64/32/16비트 정수를 의미

```
struct exfat_dentry {
	__u8 type;
	union {
		struct {
			__u8 num_ext;
			__le16 checksum;
			__le16 attr;
			__le16 reserved1;
			__le16 create_time;
			__le16 create_date;
			__le16 modify_time;
			__le16 modify_date;
			__le16 access_time;
			__le16 access_date;
			__u8 create_time_cs;
			__u8 modify_time_cs;
			__u8 create_tz;
			__u8 modify_tz;
			__u8 access_tz;
			__u8 reserved2[7];
		} __packed file; /* file directory entry */
		struct {
			__u8 flags;
			__u8 reserved1;
			__u8 name_len;
			__le16 name_hash;
			__le16 reserved2;
			__le64 valid_size;
			__le32 reserved3;
			__le32 start_clu;
			__le64 size;
		} __packed stream; /* stream extension directory entry */
		struct {
			__u8 flags;
			__le16 unicode_0_14[EXFAT_FILE_NAME_LEN];
		} __packed name; /* file name directory entry */
		struct {
			__u8 flags;
			__u8 reserved[18];
			__le32 start_clu;
			__le64 size;
		} __packed bitmap; /* allocation bitmap directory entry */
		struct {
			__u8 reserved1[3];
			__le32 checksum;
			__u8 reserved2[12];
			__le32 start_clu;
			__le64 size;
		} __packed upcase; /* up-case table directory entry */
		struct {
			__u8 flags;
			__u8 vendor_guid[16];
			__u8 vendor_defined[14];
		} __packed vendor_ext; /* vendor extension directory entry */
		struct {
			__u8 flags;
			__u8 vendor_guid[16];
			__u8 vendor_defined[2];
			__le32 start_clu;
			__le64 size;
		} __packed vendor_alloc; /* vendor allocation directory entry */
		struct {
			__u8 flags;
			__u8 custom_defined[18];
			__le32 start_clu;
			__le64 size;
		} __packed generic_secondary; /* generic secondary directory entry */
	} __packed dentry;
} __packed;
```
커널에서 사용하는 모든 디렉터리 엔트리 타입을 정의하고 있음.
**Primary Entries**
- file: 파일/디렉터리 엔트리 (type: 0x85)
- bitmap: allocation bitmap 엔트리 (type: 0x81)
- upcase: upcase table 엔트리 (type: 0x82)
- vendor_ext: 벤더 확장 엔트리 (type: 0xE0)

**Secondary Entries**
- stream: stream extension 엔트리 (type: 0xC0) - 파일 크기, 시작 클러스터 정보
- name: 파일 이름 엔트리 (type: 0xC1) - 파일명을 유니코드로
- vendor_alloc: 벤더 할당 엔트리 (type: 0xE1)
- generic_secondary: 일반 보조 엔트리

### exfat_fs.h
리눅스 커널이 exFAT을 처리하기 위해 필요한 메모리 구조체들을 정의하고 있음. 

```
/*
 * EXFAT file system superblock in-memory data
 */
struct exfat_sb_info {
	unsigned long long num_sectors; /* num of sectors in volume */
	unsigned int num_clusters; /* num of clusters in volume */
	unsigned int cluster_size; /* cluster size in bytes */
	unsigned int cluster_size_bits;
	unsigned int sect_per_clus; /* cluster size in sectors */
	unsigned int sect_per_clus_bits;
	unsigned long long FAT1_start_sector; /* FAT1 start sector */
	unsigned long long FAT2_start_sector; /* FAT2 start sector */
	unsigned long long data_start_sector; /* data area start sector */
	unsigned int num_FAT_sectors; /* num of FAT sectors */
	unsigned int root_dir; /* root dir cluster */
	unsigned int dentries_per_clu; /* num of dentries per cluster */
	unsigned int vol_flags; /* volume flags */
	unsigned int vol_flags_persistent; /* volume flags to retain */
	struct buffer_head *boot_bh; /* buffer_head of BOOT sector */

	unsigned int map_clu; /* allocation bitmap start cluster */
	unsigned int map_sectors; /* num of allocation bitmap sectors */
	struct buffer_head **vol_amap; /* allocation bitmap */

	unsigned short *vol_utbl; /* upcase table */

	unsigned int clu_srch_ptr; /* cluster search pointer */
	unsigned int used_clusters; /* number of used clusters */

	unsigned long s_exfat_flags; /* Exfat superblock flags */

	struct mutex s_lock; /* superblock lock */
	struct mutex bitmap_lock; /* bitmap lock */
	struct exfat_mount_options options;
	struct nls_table *nls_io; /* Charset used for input and display */
	struct ratelimit_state ratelimit;

	spinlock_t inode_hash_lock;
	struct hlist_head inode_hashtable[EXFAT_HASH_SIZE];
	struct rcu_head rcu;
};
```
먼저 **super block** 을 알아보자. 리눅스 커널 VFS에서 사용하는 용어로 마운트된 파일 시스템의 전체 정보를 담는 메모리 구조체라고 보면 될 거 같다. 즉, 부트 섹터를 파싱한 정보를 커널이 쓰기 편하게 재구성한 구조체라고 할 수 있다. 일부 필드만 알아보도록 하겠다. 

#### 볼륨 기본 정보
- num_sectors: 전체 볼륨의 섹터 수
- num_clusters: 전체 클러스터 수
- cluster_size: 클러스터의 크기 (바이트 단위)
- cluster_size_bits: 클러스터 크기의 비트 쉬프트 값
- sect_per_clus: 클러스터 당 섹터 수
- sect_per_clus_bits: 클러스터 당 섹터 수의 비트 쉬프트 값

#### 영역별 시작 위치
- FAT1_start_sector: First FAT 테이블 시작 섹터
- FAT2_start_sector: Second FAT 테이블 시작 섹터 (백업)
- data_start_sector: 실제 데이터 영역 시작 섹터
- num_FAT_sectors: FAT 테이블 크기 (섹터 단위)

#### 루트 디렉터리 및 엔트리
- root_dir: 루트 디렉터리의 클러스터 번호
- dentries_per_clu: 클러스터 당 디렉터리 엔트리 개수
- vol_flags: 현재 볼륨 플래그
- vol_flags_persistent: 유지해야 할 볼륨 플래그

#### Allocation Bitmap
- map_clu: Allocation Bitmap이 저장된 시작 클러스터
- map_sectors: Allocation Bitmap의 섹터 수
- **vol_amap: Allocation Bitmap의 버퍼 배열

#### Up case table
- *vol_utbl: 업케이스 테이블의 메모리 상 위치

## 파일 시스템 초기화 및 디스크 마운트
### init_exfat_fs
`insmod exfat.ko`나 커널이 부팅될 때 실행되는 함수로, exFAT 파일 시스템을 Linux VFS에 등록
```
module_init(init_exfat_fs);
```
#### 1. 클러스터 캐시 초기화
```
err = exfat_cache_init();
if (err)
	return err;
```
`cache.c`에 정의된 함수를 호출하여 exFAT 클러스터 캐시 시스템을 초기화해준다. 그런데 위에서 확인한 exFAT 명세에는 캐시에 대한 언급이 없었는데 캐시를 초기화하고 있어 cache.c도 잠깐 분석해보았다. 

```
// SPDX-License-Identifier: GPL-2.0-or-later
/*
 *  linux/fs/fat/cache.c
 *
 *  Written 1992,1993 by Werner Almesberger
 *
 *  Mar 1999. AV. Changed cache, so that it uses the starting cluster instead
 *	of inode number.
 *  May 1999. AV. Fixed the bogosity with FAT32 (read "FAT28"). Fscking lusers.
 *  Copyright (C) 2012-2013 Samsung Electronics Co., Ltd.
 */
 ```
fat 코드를 수정해서 사용하고 있었다. 이 코드들은 exFAT 명세에는 없지만 커널의 성능 최적화를 위한 구현이다. exFAT에서 각종 데이터는 클러스터 체인으로 저장이 되는데, FAT를 따라서 선형 탐색으로 찾아야 하는 구조여서 매번 디스크 I/O가 발생하게 된다. 

```
struct exfat_cache {
	struct list_head cache_list;
	unsigned int nr_contig;	/* number of contiguous clusters */
	unsigned int fcluster;	/* cluster number in the file. */
	unsigned int dcluster;	/* cluster number on disk. */
};
```
**exfat_cache** 구조체는 위와 같다. 하나씩 살펴보자
- list_head cache_list: 캐시 엔트리들을 연결하기 위한 **연결 리스트 포인터** 이다. 이 포인터를 통해 여러 `exfat_cache` 구조체들이 연결되어 **LRU** 방식으로 관리가 된다.
- fcluster: 파일 내의 **논리적인 클러스터 시작 번호**. 파일의 관점에서 본 데이터 블록의 순번이다. 
- dcluster: 디스크 상에서의 **물리적인 클러스터 시작 번호**. fcluster가 100이고 dcluster가 5000이면, 파일의 101번째 데이터 블록이 디스크의 5001 번째 슬롯에 있다는 뜻
- nr_contig: fcluster부터 시작되는 **연속 할당된 클러스터 개수**. exFAT은 많은 경우 여러 클러스터를 붙여서 연속 할당을 하는데, 캐시에서 이를 반영한 것 같다. 

```
int exfat_cache_init(void)
{
	exfat_cachep = kmem_cache_create("exfat_cache",
				sizeof(struct exfat_cache),
				0, SLAB_RECLAIM_ACCOUNT,
				exfat_cache_init_once);
	if (!exfat_cachep)
		return -ENOMEM;
	return 0;
}
```

exFAT 캐시 시스템 전체가 사용할 공간(**전용 메모리 풀**)을 마련하는 작업을 해주고 있다. kmalloc과 다르게 kmem_cache_create의 경우 이미 `exfat_cache` 구조체의 크기에 맞는 공간들을 마련해두고 있어서, 탐색 과정 없이 빠르게 할당과 해제를 할 수 있다는 점에서 특징을 갖는 것 같다. 또한 전용 풀에서 할당한 exfat_cache 객체들은 물리적 메모리 상으로도 가깝게 있을 확률이 높아, 한번 캐시에 올라오면 다른 객체들도 캐시에 올라와 있을 확률이 높다. 

#### 2. 아이노드 캐시 초기화
```
exfat_inode_cachep = kmem_cache_create("exfat_inode_cache",
			sizeof(struct exfat_inode_info),
			0, SLAB_RECLAIM_ACCOUNT,
			exfat_inode_init_once);
if (!exfat_inode_cachep) {
	err = -ENOMEM;
	goto shutdown_cache;
}
```

아까 클러스터 캐시랑 다르게 이건 전체 exFAT 파일 시스템에서 공유하는 메모리 풀로 **exfat inode** 를 아래와 같이 빠르게 할당하고 해제하기 위해 사용하는 캐시이다. 참고로 클러스터 캐시는 `struct exfat_inode_info`에서 확인할 수 있다.
```
static struct inode *exfat_alloc_inode(struct super_block *sb)
{
	struct exfat_inode_info *ei;

	ei = alloc_inode_sb(sb, exfat_inode_cachep, GFP_NOFS);
	if (!ei)
		return NULL;

	init_rwsem(&ei->truncate_lock);
	return &ei->vfs_inode;
}

static void exfat_free_inode(struct inode *inode)
{
	kmem_cache_free(exfat_inode_cachep, EXFAT_I(inode));
}
```

#### 3. 파일 시스템 등록
```
err = register_filesystem(&exfat_fs_type);
if (err)
	goto destroy_cache;
```
VFS에 exFAT 파일 시스템을 등록하는 과정이다.

### exfat_init_fs_context
파일 시스템을 초기화하고 VFS에 exFAT 타입을 등록하였다. 이제 디스크 마운트를 해보자

```
mount -t exfat /dev/sdb1 /mnt/usb
```
이런 커맨드가 호출이 된다고 하자. VFS 레벨에서는 `exFAT` 타입을 찾고, 새로운 fs_context 구조체를 만든 다음에 **exfat_fs_type.init_fs_context** 를 호출한다. 

```
sbi = kzalloc(sizeof(struct exfat_sb_info), GFP_KERNEL);
if (!sbi)
    return -ENOMEM;
```
- kzalloc: 커널 메모리를 할당하고 0으로 초기화
- GFP_KERNEL: 슬립 가능한 일반 커널 메모리
커널에서 사용할 수 있도록 exFAT의 메타데이터를 담을 슈퍼 블록 구조체를 할당, 초기화하고 있다.

```
mutex_init(&sbi->s_lock);
mutex_init(&sbi->bitmap_lock);
ratelimit_state_init(&sbi->ratelimit, DEFAULT_RATELIMIT_INTERVAL,
		DEFAULT_RATELIMIT_BURST);
```
- s_lock: 슈퍼블록 보호 뮤텍스
- bitmap_lock: 클러스터를 할당하고 해제할 때 bitmap을 보호하는 뮤텍스이다.

```
sbi->options.fs_uid = current_uid();
sbi->options.fs_gid = current_gid();
sbi->options.fs_fmask = current->fs->umask;
sbi->options.fs_dmask = current->fs->umask;
sbi->options.allow_utime = -1;
sbi->options.iocharset = exfat_default_iocharset;
sbi->options.errors = EXFAT_ERRORS_RO;
```
기본 마운트 옵션을 설정하고 있다. 현재 마운트하는 사용자의 UID/GID, 현재 프로세스의 umask 값 등을 부여하고 있다.

```
fc->s_fs_info = sbi;
fc->ops = &exfat_context_ops;
```
- fc->s_fs_info=sbi: 파일 시스템 컨텍스트에 sbi를 연결
- fc->ops = &exfat_context_ops: 다음 단계 함수들을 설정해주고 있다.

```
static const struct fs_context_operations exfat_context_ops = {
	.parse_param	= exfat_parse_param,
	.get_tree	= exfat_get_tree,
	.free		= exfat_free,
	.reconfigure	= exfat_reconfigure,
};
```
- exfat_parse_param: 마운트 옵션 파싱
- exfat_get_tree: 실제 마운트 수행
- exfat_free: 컨텍스트 해제
- exfat_reconfigure: 재마운트

### exfat_parse_param
아까 봤던 exfat_fs.h에는 마운트 옵션을 담기 위한 구조체가 있다.
```
/*
 * exfat mount in-memory data
 */
struct exfat_mount_options {
	kuid_t fs_uid;
	kgid_t fs_gid;
	unsigned short fs_fmask;
	unsigned short fs_dmask;
	/* permission for setting the [am]time */
	unsigned short allow_utime;
	/* charset for filename input/display */
	char *iocharset;
	/* on error: continue, panic, remount-ro */
	enum exfat_error_mode errors;
	unsigned utf8:1, /* Use of UTF-8 character set */
		 sys_tz:1, /* Use local timezone */
		 discard:1, /* Issue discard requests on deletions */
		 keep_last_dots:1; /* Keep trailing periods in paths */
	int time_offset; /* Offset of timestamps from UTC (in minutes) */
	/* Support creating zero-size directory, default: false */
	bool zero_size_dir;
};
```
exfat_parse_param 함수는 사용자가 지정한 마운트 옵션을 이 구조체에 저장하고, 실제 마운트에서 사용한다. 사용자가 따로 지정해주지 않으면 아까 기본적으로 넣어준 옵션을 그대로 사용하는 것 같다. 

### exfat_get_tree
```
static int exfat_get_tree(struct fs_context *fc)
{
	return get_tree_bdev(fc, exfat_fill_super);
}
```
생각보다 단순하게 생겼다. exFAT은 파일시스템별 로직만 제공하고 복잡한 과정은 VFS에서 처리하기 때문이다. 
- get_tree_bdev: VFS의 **블록 디바이스용 공통 마운트 함수**. 블록 디바이스는 HDD, SSD 같은 물리 저장매체라고 보면 될 거 같다. 

### get_tree_bdev
```
/**
 * get_tree_bdev - Get a superblock based on a single block device
 * @fc: The filesystem context holding the parameters
 * @fill_super: Helper to initialise a new superblock
 */
int get_tree_bdev(struct fs_context *fc,
		int (*fill_super)(struct super_block *,
				  struct fs_context *))
{
	return get_tree_bdev_flags(fc, fill_super, 0);
}
EXPORT_SYMBOL(get_tree_bdev);
```
매개변수부터 보자
- struct fs_context *fc: 파일 시스템 컨텍스트, 마운트에 필요한 정보 갖고 있음
- fill_super: 함수 포인터, 파일 시스템별 초기화 함수가 전달되는데 exFAT은 **exfat_fill_super**

```
/*
 * Filesystem context for holding the parameters used in the creation or
 * reconfiguration of a superblock.
 *
 * Superblock creation fills in ->root whereas reconfiguration begins with this
 * already set.
 *
 * See Documentation/filesystems/mount_api.rst
 */
struct fs_context {
	const struct fs_context_operations *ops;
	struct mutex		uapi_mutex;	/* Userspace access mutex */
	struct file_system_type	*fs_type;
	void			*fs_private;	/* The filesystem's context */
	void			*sget_key;
	struct dentry		*root;		/* The root and superblock */
	struct user_namespace	*user_ns;	/* The user namespace for this mount */
	struct net		*net_ns;	/* The network namespace for this mount */
	const struct cred	*cred;		/* The mounter's credentials */
	struct p_log		log;		/* Logging buffer */
	const char		*source;	/* The source name (eg. dev path) */
	void			*security;	/* LSM options */
	void			*s_fs_info;	/* Proposed s_fs_info */
	unsigned int		sb_flags;	/* Proposed superblock flags (SB_*) */
	unsigned int		sb_flags_mask;	/* Superblock flags that were changed */
	unsigned int		s_iflags;	/* OR'd with sb->s_iflags */
	enum fs_context_purpose	purpose:8;
	enum fs_context_phase	phase:8;	/* The phase the context is in */
	bool			need_free:1;	/* Need to call ops->free() */
	bool			global:1;	/* Goes into &init_user_ns */
	bool			oldapi:1;	/* Coming from mount(2) */
	bool			exclusive:1;    /* create new superblock, reject existing one */
};
```
이게 `fs_context` 구조체다. 아까 봤었던 **fc->ops = &exfat_context_ops** 같은 할당들이 전부 fs_context 구조체를 업데이트 하는 과정이다. 필드 몇개만 살펴보자면
- fs_context_operations *ops: 컨텍스트 연산 함수들
- *s_fs_info: 슈퍼블록 데이터
- struct cred *cred: 마운트 수행자의 credential
- sb_flags: 슈퍼블록 플래그 
이런 필드들이 있다.

다시 함수로 돌아가서 get_tree_bdev 함수는 플래그를 0으로 고정하여 **get_tree_bdev_flags** 를 다시 호출해주고 있다. 단순한 wrapper의 기능을 하는 것으로 보인다.

### get_tree_bdev_flags
```
if (!fc->source)
	return invalf(fc, "No source specified");
```
- fc->source: `/dev/sdb1` 같은 디바이스 경로를 의미한다. 만약 경로가 없다면 바로 에러를 반환하는 로직이다.

```
error = lookup_bdev(fc->source, &dev);
if (error) {
	if (!(flags & GET_TREE_BDEV_QUIET_LOOKUP))
		errorf(fc, "%s: Can't lookup blockdev", fc->source);
	return error;
}
```
디바이스 경로를 바탕으로 블록 디바이스를 찾는 과정이다. lookup_bdev 함수의 경우, 주어진 경로를 바탕으로 디바이스를 찾으면 **dev_t dev** 형태의 포인터를 반환한다. 

```
s = sget_dev(fc, dev);
if (IS_ERR(s))
	return PTR_ERR(s);
```
**sget_dev** 함수는 디바이스 번호를 바탕으로 기존 슈퍼 블록을 찾거나 새로 생성하는 함수다. 두 가지 경우가 있다.
- 기존 슈퍼블록 검색: 기존에 있던 슈퍼블록을 반환하는 경우로, 이 블록은 이미 초기화가 완료된 상태라 **s->s_root != NULL** 이다. 
- 새로운 슈퍼블록 생성: 새로운 슈퍼블록을 생성하는 경우로, 이 경우에는 **s->s_root == NULL** 이다. 

```
else {
	error = setup_bdev_super(s, fc->sb_flags, fc);
	if (!error)
		error = fill_super(s, fc);
	if (error) {
		deactivate_locked_super(s);
		return error;
	}
	s->s_flags |= SB_ACTIVE;
}
```
새로운 슈퍼블록을 생성한 경우만 살펴보도록 하겠다. 생성된 슈퍼블록을 초기화하는 단계라고 보면 된다. 지금까지의 상황을 정리하면 아래와 같다. 
1. exfat_init_fs_context: **exfat_sb_info** 메모리를 할당한다. 이 구조체는 exFAT 전용 메타데이터 구조체라고 보면 된다.
2. exfat_get_tree
3. get_tree_bdev_flags
4. lookup_bdev: 여기서 `dev_t` 포인터를 반환해준다.
5. sget_dev(fc, dev): 디바이스 번호를 바탕으로 슈퍼블록이 원래 만들어둔게 있는지, 새로 만들어야 하는지에 따라 슈퍼 블록을 반환한다.
6. sget_fc: 새로운 슈퍼 블록 반환

```
if (!s) {
	spin_unlock(&sb_lock);
	s = alloc_super(fc->fs_type, fc->sb_flags, user_ns);
	if (!s)
		return ERR_PTR(-ENOMEM);
	goto retry;
}

s->s_fs_info = fc->s_fs_info;
```
sget_fc 함수의 일부분인데, 새로운 슈퍼 블록을 만들어주는 경우이다. **s->s_fs_info = fc->s_sfs_info** 이 부분에서 새로 만든 슈퍼블록이랑 이전에 만들어뒀던 `exfat_sb_info`를 연결해주고 있다. 다시 setup_bdev_super로 돌아가자.

### setup_bdev_super
생성한 슈퍼블록과 실제 블록 디바이스를 연결하여 **I/O 가능한 상태** 로 만드는 과정이다. 

```
blk_mode_t mode = sb_open_mode(sb_flags);

...

/*
 * Return the correct open flags for blkdev_get_by_* for super block flags
 * as stored in sb->s_flags.
 */
#define sb_open_mode(flags) \
	(BLK_OPEN_READ | BLK_OPEN_RESTRICT_WRITES | \
	 (((flags) & SB_RDONLY) ? 0 : BLK_OPEN_WRITE))
```
- BLK_OPEN_READ, BLK_OPEN_RESTRICT_WRITES: 읽기 권한과 직접 쓰기 제한 플래그는 항상 포함된다. 후자는 아마 디바이스에 대한 직접 쓰기를 방지하려는 의도 같다.
- 뒷 부분은 Readonly 여부에 따라 쓰기 플래그를 결정하고 있다

```
bdev_file = bdev_file_open_by_dev(sb->s_dev, mode, sb, &fs_holder_ops);
if (IS_ERR(bdev_file)) {
	if (fc)
		errorf(fc, "%s: Can't open blockdev", fc->source);
	return PTR_ERR(bdev_file);
}
bdev = file_bdev(bdev_file);
```
**bdev_file_open_by_dev**: 블록 디바이스를 파일 형태로 여는 함수다. **dev_t** 디바이스 번호로 블록 디바이스를 찾아서 파일 핸들을 반환한다. 
```
bdev_file = alloc_file_pseudo_noaccount(BD_INODE(bdev),
			blockdev_mnt, "", flags | O_LARGEFILE, &def_blk_fops);
```
bdev_file은 실제 파일이 아닌 **가상 파일**, **메모리상 가상 객체** 이다. dev_t로 지정된 블록 디바이스에 파일 인터페이스로 접근할 수 있도록 만든 가상 파일 객체로 실제 물리 디바이스에 연결되어 있다고 생각하면 된다. `bdev`는 파일 객체에서 블록 디바이스 구조체를 추출한 것이다. 

```
sb->s_bdev_file = bdev_file;
sb->s_bdev = bdev;
sb->s_bdi = bdi_get(bdev->bd_disk->bdi);
if (bdev_stable_writes(bdev))
	sb->s_iflags |= SB_I_STABLE_WRITES;
spin_unlock(&sb_lock);

snprintf(sb->s_id, sizeof(sb->s_id), "%pg", bdev);
shrinker_debugfs_rename(sb->s_shrink, "sb-%s:%s", sb->s_type->name,
			sb->s_id);
sb_set_blocksize(sb, block_size(bdev));
```
이제 슈퍼블록을 설정하는 마지막 단계다. 
- s_bdev_file: 파일 인터페이스로 아까 얻은 **디바이스 파일 핸들** 을 저장
- s_bdev: 블록 디바이스에 직접 접근하기 위한 필드로 블록 디바이스 구조체를 저장
- s_bdi: I/O 최적화 정보, Backing device info를 설정
- snprintf(sb->s_id, sizeof(sb->s_id), "%pg", bdev): 슈퍼블록 ID를 생성함

이제 슈퍼블록이 실제 블록 디바이스와 완전히 연결이 되었다. 드디어 **fill_super** 함수를 호출할 차례이다. 

### exfat_fill_super
`exfat_fill_super(외부)`, `__exfat_fill_super(내부)` 두 함수가 있는데, 내부 함수에서 exFAT 특화된 로직을 처리해주고 있기에 내부 함수를 분석해보겠다.

#### exfat_read_boot_sector
```
/* set block size to read super block */
sb_min_blocksize(sb, 512);

/* read boot sector */
sbi->boot_bh = sb_bread(sb, 0);
```
부트 섹터는 **섹터 0** 에 있다. 최소 블록 크기 512바이트를 설정한 다음에 부트 섹터를 읽어들이는 과정이다. 

```
/* check the validity of BOOT */
if (le16_to_cpu((p_boot->signature)) != BOOT_SIGNATURE) {
	exfat_err(sb, "invalid boot record signature");
	return -EINVAL;
}

if (memcmp(p_boot->fs_name, STR_EXFAT, BOOTSEC_FS_NAME_LEN)) {
	exfat_err(sb, "invalid fs_name"); /* fs_name may unprintable */
	return -EINVAL;
}

/*
 * must_be_zero field must be filled with zero to prevent mounting
 * from FAT volume.
 */
if (memchr_inv(p_boot->must_be_zero, 0, sizeof(p_boot->must_be_zero)))
	return -EINVAL;

if (p_boot->num_fats != 1 && p_boot->num_fats != 2) {
	exfat_err(sb, "bogus number of FAT structure");
	return -EINVAL;
}

/*
 * sect_size_bits could be at least 9 and at most 12.
 */
if (p_boot->sect_size_bits < EXFAT_MIN_SECT_SIZE_BITS ||
    p_boot->sect_size_bits > EXFAT_MAX_SECT_SIZE_BITS) {
	exfat_err(sb, "bogus sector size bits : %u",
			p_boot->sect_size_bits);
	return -EINVAL;
}

/*
 * sect_per_clus_bits could be at least 0 and at most 25 - sect_size_bits.
 */
if (p_boot->sect_per_clus_bits > EXFAT_MAX_SECT_PER_CLUS_BITS(p_boot)) {
	exfat_err(sb, "bogus sectors bits per cluster : %u",
			p_boot->sect_per_clus_bits);
	return -EINVAL;
}
```
- BOOT_SIGNATURE: 0xAA55 확인(**le16_to_cpu**: 16비트 리틀 엔디안 값을 CPU 바이트 순서로 변환)
- fs_name: "EXFAT   " 문자열이 맞는지 확인
- must_be_zero: 이 필드는 전부 0이여야 함
- num_fats: exFAT은 1개 또는 2개의 FAT만 허용함. 1이나 2가 아니면 마운트 거부
- sect_size_bits: 섹터 크기는 512KB~ 4MB
- sect_per_clus_bits: 클러스터 당 섹터는 최대 25-`sec_size_bits`

```
sbi->sect_per_clus = 1 << p_boot->sect_per_clus_bits;
sbi->sect_per_clus_bits = p_boot->sect_per_clus_bits;
sbi->cluster_size_bits = p_boot->sect_per_clus_bits +
	p_boot->sect_size_bits;
sbi->cluster_size = 1 << sbi->cluster_size_bits;
sbi->num_FAT_sectors = le32_to_cpu(p_boot->fat_length);
sbi->FAT1_start_sector = le32_to_cpu(p_boot->fat_offset);
sbi->FAT2_start_sector = le32_to_cpu(p_boot->fat_offset);
if (p_boot->num_fats == 2)
	sbi->FAT2_start_sector += sbi->num_FAT_sectors;
sbi->data_start_sector = le32_to_cpu(p_boot->clu_offset);
sbi->num_sectors = le64_to_cpu(p_boot->vol_length);
/* because the cluster index starts with 2 */
sbi->num_clusters = le32_to_cpu(p_boot->clu_count) +
	EXFAT_RESERVED_CLUSTERS;

sbi->root_dir = le32_to_cpu(p_boot->root_cluster);
sbi->dentries_per_clu = 1 <<
	(sbi->cluster_size_bits - DENTRY_SIZE_BITS);

sbi->vol_flags = le16_to_cpu(p_boot->vol_flags);
sbi->vol_flags_persistent = sbi->vol_flags & (VOLUME_DIRTY | MEDIA_FAILURE);
sbi->clu_srch_ptr = EXFAT_FIRST_CLUSTER;
```

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

- **sbi->sect_per_clus**: `SectorsPerClusterShift`를 이용해서 계산 (1 << bits)
- **sbi->sect_per_clus_bits**: `SectorsPerClusterShift`를 그대로 복사
- **sbi->cluster_size_bits**: `BytesPerSectorShift + SectorsPerClusterShift` 계산
- **sbi->cluster_size**: `cluster_size_bits`를 이용해서 계산 (1 << bits)

- **sbi->num_FAT_sectors**: `FatLength`를 LE 변환해서 복사
- **sbi->FAT1_start_sector**: `FatOffset`을 LE 변환해서 복사
- **sbi->FAT2_start_sector**: `FatOffset + (NumberOfFats==2 ? FatLength : 0)` 조건부 계산

- **sbi->data_start_sector**: `ClusterHeapOffset`을 LE 변환해서 복사
- **sbi->num_sectors**: `VolumeLength`를 LE 변환해서 복사
- **sbi->num_clusters**: `ClusterCount + EXFAT_RESERVED_CLUSTERS(2)` 계산

- **sbi->root_dir**: `FirstClusterOfRootDirectory`를 LE 변환해서 복사
- **sbi->dentries_per_clu**: `cluster_size_bits - DENTRY_SIZE_BITS`를 이용해서 계산 (1 << result)
- **sbi->vol_flags**: `VolumeFlags`를 LE 변환해서 복사
- **sbi->vol_flags_persistent**: `vol_flags & (VOLUME_DIRTY | MEDIA_FAILURE)` 마스킹
- **sbi->clu_srch_ptr**: `EXFAT_FIRST_CLUSTER` 상수로 초기화

```
/* check consistencies */
if ((u64)sbi->num_FAT_sectors << p_boot->sect_size_bits <
	(u64)sbi->num_clusters * 4) {
	exfat_err(sb, "bogus fat length");
	return -EINVAL;
}
```
exFAT에서 각 클러스터는 FAT에서 4바이트를 차지
- Left: 실제 FAT 크기. `num_FAT_sectors`는 FAT가 차지하는 섹터 수. 이걸 `sect_size_bits`만큼 시프트 레프트 하면 결국 FAT 테이블의 실제 크기를 의미
- Right: 클러스터 당 4바이트 x 클러스터 개수 => 모든 클러스터를 표현하는데 필요한 최소 바이트

즉, 실제 FAT 크기 > 필요한 최소 크기 인지를 확인하는 부분

```
if (sbi->data_start_sector <
	(u64)sbi->FAT1_start_sector +
	(u64)sbi->num_FAT_sectors * p_boot->num_fats) {
	exfat_err(sb, "bogus data start sector");
	return -EINVAL;
}
```
- Left: 데이터 영역 시작 위치
- Right: FAT 영역의 끝 위치(FAT 개수 2개인 경우도 처리 가능)
두 영역이 겹치면 데이터 영역과 FAT가 겹친다는 뜻!!

```
sb->s_maxbytes = (u64)(sbi->num_clusters - EXFAT_RESERVED_CLUSTERS) <<
	sbi->cluster_size_bits;
```
- `num_clusters - 2`: 사용 가능한 클러스터 수(0, 1은 제외)
- `<< cluster_size_bits`: 클러스터를 바이트로 변환
이론적으로 가능한 파일의 최대 크기를 알려주는 과정

#### exfat_verify_boot_region
```
u32 exfat_calc_chksum32(void *data, int len, u32 chksum, int type)
{
	int i;
	u8 *c = (u8 *)data;

	for (i = 0; i < len; i++, c++) {
		if (unlikely(type == CS_BOOT_SECTOR &&
			     (i == 106 || i == 107 || i == 112)))
			continue;
		chksum = ((chksum << 31) | (chksum >> 1)) + *c;
	}
	return chksum;
}
```
`fs/exfat/misc.c`에 있는 체크섬 계산 알고리즘이다. 
- u32 chksum: 32비트 체크섬 변수
데이터를 한바이트씩 읽으면서, 이전에 계산된 체크섬 값을 오른쪽으로 한 칸 돌리고, 거기에 현재 읽은 바이트 값을 더하는 계산을 반복하고 있다. 중간에 continue 하는 부분은 checksum 계산에서 제외되는 `PercentInUse`와 `VolumeFlags` 부분이다. 

```
for (sn = 0; sn < 11; sn++) {
	bh = sb_bread(sb, sn);
	if (!bh)
		return -EIO;

	if (sn != 0 && sn <= 8) {
		/* extended boot sector sub-regions */
		p_sig = (__le32 *)&bh->b_data[sb->s_blocksize - 4];
		if (le32_to_cpu(*p_sig) != EXBOOT_SIGNATURE)
			exfat_warn(sb, "Invalid exboot-signature(sector = %d): 0x%08x",
					sn, le32_to_cpu(*p_sig));
	}

	chksum = exfat_calc_chksum32(bh->b_data, sb->s_blocksize,
		chksum, sn ? CS_DEFAULT : CS_BOOT_SECTOR);
	brelse(bh);
}
```
체크섬을 계산하는 부분이다. Main Boot Sector의 0번부터 10번 섹터까지 순회를 하면서, 각 섹터마다 읽고 체크섬 계산을 반복시키면서 누적한다. 계산하면서 extended 영역에 대한 시그니처 검사도 같이 진행한다. 물론 시그니처가 틀려도 바로 실패로 빠지지는 않는다.

```
/* boot checksum sub-regions */
bh = sb_bread(sb, sn);
if (!bh)
	return -EIO;

for (i = 0; i < sb->s_blocksize; i += sizeof(u32)) {
	p_chksum = (__le32 *)&bh->b_data[i];
	if (le32_to_cpu(*p_chksum) != chksum) {
		exfat_err(sb, "Invalid boot checksum (boot checksum : 0x%08x, checksum : 0x%08x)",
				le32_to_cpu(*p_chksum), chksum);
		brelse(bh);
		return -EINVAL;
	}
}
```
이제는 11번 섹터를 읽어온 다음(체크섬 섹터), 계산한 값과 대조해가면서 섹터 전체의 값들이 완벽하게 일치하는지 확인한다. 

#### exfat_create_upcase_table
```
clu.dir = sbi->root_dir;
clu.flags = ALLOC_FAT_CHAIN;
```
먼저 루트 디렉터리를 초기화 해주고 있다.

```
while (clu.dir != EXFAT_EOF_CLUSTER) {
	for (i = 0; i < sbi->dentries_per_clu; i++) {
		ep = exfat_get_dentry(sb, &clu, i, &bh);
		if (!ep)
			return -EIO;

		type = exfat_get_entry_type(ep);
		if (type == TYPE_UNUSED) {
			brelse(bh);
			break;
		}

		if (type != TYPE_UPCASE) {
			brelse(bh);
			continue;
		}

		tbl_clu  = le32_to_cpu(ep->dentry.upcase.start_clu);
		tbl_size = le64_to_cpu(ep->dentry.upcase.size);

		sector = exfat_cluster_to_sector(sbi, tbl_clu);
		num_sectors = ((tbl_size - 1) >> blksize_bits) + 1;
		ret = exfat_load_upcase_table(sb, sector, num_sectors,
			le32_to_cpu(ep->dentry.upcase.checksum));

		brelse(bh);
		if (ret && ret != -EIO) {
			/* free memory from exfat_load_upcase_table call */
			exfat_free_upcase_table(sbi);
			goto load_default;
		}

		/* load successfully */
		return ret;
	}

	if (exfat_get_next_cluster(sb, &(clu.dir)))
		return -EIO;
}
```
위의 반복문은 **루트 디렉터리의 클러스터 체인 전체** 를 순회하고 있다. 외부 루프는 클러스터 단위로 돌고 있고, 내부 루프에서는 엔트리 단위로 돌고 있다. 루트 디렉토리의 클러스터 체인의 모든 클러스터의 모든 엔트리를 확인해서 **upcase entry** 를 찾고 있다. 

그리고 upcase 테이블을 찾으면 메모리에 로드한다.
```
tbl_clu  = le32_to_cpu(ep->dentry.upcase.start_clu);
tbl_size = le64_to_cpu(ep->dentry.upcase.size);

sector = exfat_cluster_to_sector(sbi, tbl_clu);
num_sectors = ((tbl_size - 1) >> blksize_bits) + 1;
ret = exfat_load_upcase_table(sb, sector, num_sectors,
    le32_to_cpu(ep->dentry.upcase.checksum));
```

#### exfat_cound_used_clusters
```
int exfat_count_used_clusters(struct super_block *sb, unsigned int *ret_count)
{
	struct exfat_sb_info *sbi = EXFAT_SB(sb);
	unsigned int count = 0;
	unsigned int i, map_i = 0, map_b = 0;
	unsigned int total_clus = EXFAT_DATA_CLUSTER_COUNT(sbi);
	unsigned int last_mask = total_clus & (BITS_PER_LONG - 1);
	unsigned long *bitmap, clu_bits;

	total_clus &= ~last_mask;
	for (i = 0; i < total_clus; i += BITS_PER_LONG) {
		bitmap = (void *)(sbi->vol_amap[map_i]->b_data + map_b);
		count += hweight_long(*bitmap);
		map_b += sizeof(long);
		if (map_b >= (unsigned int)sb->s_blocksize) {
			map_i++;
			map_b = 0;
		}
	}

	if (last_mask) {
		bitmap = (void *)(sbi->vol_amap[map_i]->b_data + map_b);
		clu_bits = lel_to_cpu(*(__le_long *)bitmap);
		count += hweight_long(clu_bits & BITMAP_LAST_WORD_MASK(last_mask));
	}

	*ret_count = count;
	return 0;
}
```
Allocation Bitmap을 읽으면서 비트 카운팅을 통해 전체 파일 시스템의 사용량을 계산하는 함수이다.