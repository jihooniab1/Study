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
| Main Extended Boot Sectors | 1 | 8 |  [2](#main-and-backup-extended-boot-sectors-sub-regions)  |
| Main OEM Parameters | 9 | 1 |  [3](#main-and-backup-oem-parameters-sub-regions)  |
| Main Reserved | 10 | 1 |    |
| Main Boot Checksum | 9 | 1 |  [4](#main-and-backup-boot-checksum-sub-regions)  |
| **Backup Boot Region** |  |  |   |
| Backup Boot Sector | 12 | 1 |    |
| Backup Extended Boot Sectors | 13 | 8 |    |
| Backup OEM Parameters | 21 | 1 |    |
| Backup Reserved | 22 | 1 |    |
| Backup Boot Checksum | 23 | 1 |    |
| **FAT Region** |  |  |  [5](#file-allocation-table-region)  |
| FAT Alignment | 24 | `FatOffset - 24` |    |
| First FAT | `FatOffset` | `FatLength` |    |
| Second FAT | `FatOffset + FatLength` | `FatLength * (NumberOfFats - 1)` |    |
| **Data Region** |  |  | [6](#data-region)  |
| Cluster Heap Alignment | `FatOffset + FatLength * NumberOfFats` | `ClusterHeapOffset-(FatOffset + FatLEngth * NumberOfFats)` |    |
| Cluster Heap | `ClusterHeapOffset` | `ClusterCount * 2^SectorsPerClusterShift` |    |
| Excess Space | ClusterHeapOffset + ClusterCount * 2<sup>SectorsPerClusterShift</sup> | `VolumeLength-(ClusterHeapOffset+ClusterCount * 2^SectorsPerClusterShift)` |    |

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
Generic DirectoryEntry template에서 제공된 정의를 따름.

##### Critical Primary Directory Entries
Critical primary 디렉터리 엔트리들은 exFAT 볼륨의 적절한 관리에 중요한 정보를 포함함. 루트 디렉터리만이 critical primary 디렉터리 엔트리들을 포함함(File 디렉터리 엔트리들은 예외).

Critical primary 디렉터리 엔트리들의 정의는 주요 exFAT 개정 번호와 연관됨. 구현체들은 모든 critical primary 디렉터리 엔트리들을 지원해야 하며 이 사양이 정의하는 critical primary 디렉터리 엔트리 구조만을 기록해야 함.

##### Benign Primary Directory Entries
Benign primary 디렉터리 엔트리들은 exFAT 볼륨 관리에 유용할 수 있는 추가 정보를 포함함. 어떤 디렉터리든 benign primary 디렉터리 엔트리들을 포함할 수 있음.

Benign primary 디렉터리 엔트리들의 정의는 부 exFAT 개정 번호와 연관됨. 이 사양이나 후속 사양이 정의하는 benign primary 디렉터리 엔트리에 대한 지원은 선택사항임.

#### SecondaryCount 필드
주어진 primary 디렉터리 엔트리 바로 다음에 오는 secondary 디렉터리 엔트리의 수를 설명함. 이러한 secondary 디렉터리 엔트리들이 주어진 primary 디렉터리 엔트리와 함께 디렉터리 엔트리 세트를 구성함.

유효한 값 범위:
- **최소 0**: 이 primary 디렉터리 엔트리가 디렉터리 엔트리 세트의 유일한 엔트리임을 의미
- **최대 255**: 다음 255개의 디렉터리 엔트리와 이 primary 디렉터리 엔트리가 디렉터리 엔트리 세트를 구성함을 의미

이 템플릿에서 파생된 Critical primary 디렉터리 엔트리 구조들은 SecondaryCount와 SetChecksum 필드를 모두 재정의할 수 있음.

#### SetChecksum 필드
주어진 디렉터리 엔트리 세트의 모든 디렉터리 엔트리들의 체크섬을 포함함. 하지만 체크섬은 이 필드를 제외함. 구현체들은 주어진 디렉터리 엔트리 세트의 다른 디렉터리 엔트리를 사용하기 전에 이 필드의 내용이 유효한지 확인해야 함.

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

### Generic Secondary DirectoryEntry Template
Secondary 디렉터리 엔트리들의 중심 목적은 디렉터리 엔트리 세트에 대한 추가 정보를 제공하는 것임. Generic Secondary DirectoryEntry template을 해석하는 능력은 필수임.

Critical과 benign secondary 디렉터리 엔트리들 모두의 정의는 부 exFAT 개정 번호와 연관됨. 이 사양이나 후속 사양이 정의하는 critical 또는 benign secondary 디렉터리 엔트리에 대한 지원은 선택사항임.

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

