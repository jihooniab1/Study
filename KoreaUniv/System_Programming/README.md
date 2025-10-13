# COSE-322
https://os.korea.ac.kr/

# Index
- [1. Lecture 1](#lecture-1)
- [2. Lecture 2](#lecture-2)
- [3. Lecture 3](#lecture-3)

# Lecture 1
## File
운영체제에서 **파일** 은 저장장치를 사용자에게 쉽게 보여주기 위한 **추상화** 된 개념입니다. 운영체제에서 `File as abstraction`은 **바이트 들의 선형 배열(Linear array of bytes)** 로 정의 됩니다. 
- 선형적(Linear)은 파일 데이터가 1차원 구조로 i, i+1, i+2와 같이 순차적으로 배치되는 것을 의미합니다. 
- 배열(array)은 데이터 공간이 비어있지 않고 연속적(not sparse)이라는 것으로 프로세스의 주소 공간은 sparse 할 수 있다는 점과 차이가 있습니다.
- 파일 내에서 데이터에 접근하는 최소 단위는 **byte** 단위입니다. 물리 메모리가 `word-aligned` 되어 있더라도 파일 시스템은 바이트 단위의 접근을 보장합니다. 

파일은 주소 공간(address space)과 비교했을 때, 파일의 경우 길이가 가변적이고, 비휘발성(persistent)이며 데이터가 연속적이라는 특징이 있습니다. 반면 프로세스의 주소 공간은 길이가 고정(Fixed)되어 있으며 휘발성(Volatile)이고 비연속적일 수 있다는 차이가 있습니다. 파일과 주소 공간 모두 **프로세스 간 공유** 가 가능하다는 공통점이 있습니다.

데이터베이스에서는 failure semantics라는게 확실하게 정의되어 있음. 뭐라도 중간에 하다가 실패하면 원위치로 돌아가기(트랜잭션). 파일 시스템은 **failure semantics** 가 정의되어 있지 않음. ext4 같은 파일 시스템에서 `저널링`이라는 걸로 부분적인 보완을 하지만 DB 수준까지는 아님. 

File로 DB를 만들면 어떻게 될까? => 파일로 DB를 만들 수는 있으나 트랜잭션을 정확히 수행못할 수 있다는 단점이 있습니다.

파일 시스템은 크게 두가지 의미를 갖는데, 하나는 파일과 물리디스크 블록 간의 **mapping** 을 의미합니다. 파일 시스템은 디스크 위치 배치를 관리하고, 사용자는 스토리지의 종류나 파일이 저장된 위치를 알 필요 없습니다(**location independence**). 파일 블록 넘버가 1, 2, 3이런 파일들이 디스크 블록 번호는 108, 3010, 3011과 같은 경우를 의미합니다.

다른 하나는 디스크에 들어가 있는 파일 전체를 총칭하는 것으로 파일 시스템에 따라 파일들의 배치나 구성이 달라집니다. 프로세스와 파일 시스템간의 관계는 아래와 같이 나타낼 수 있습니다. 프로세스는 파일을 사용하는 주체로 읽기, 쓰기 작업을 수행하고, 메모리는 프로세스가 직접 접근하여 작업하는 공간입니다. 프로세스가 파일 데이터를 메모리에 적재합니다. 파일 시스템은 **파일 객체** 와 **버퍼 캐시** 로 구성되며 파일 객체는 프로세스가 접근하는 추성화된 인터페이스, 버퍼 캐시는 `디스크 I/O`를 줄이기 위한 성능 향상 메커니즘입니다.그리고 Physical Storage는 실제 데이터가 저장되는 하드웨어입니다.  <br>

![image](./images/Lec1_1.png) <br>

#### 프로세스가 Read 연산 실행
프로세스가 `read` 시스템 콜을 호출하면 파일 시스템은 먼저 **Buffer Cache** 를 확인합니다. 만약 `cache hit`라면 캐시에서 바로 메모리로 복사를 하고, `cache miss`라면 Physical Storage에서 데이터를 읽어온 다음 캐시에 저장하고, 이를 메모리로 복사합니다. 이를 통해 데이터가 프로세스의 메모리 공간으로 복사됩니다. 

#### 프로세스가 Write 연산 실행
프로세스가 `write` 시스템 콜을 호출하면 데이터가 먼저 **Buffer Cache** 에 기록된 후 나중에 Physical Storage에 기록이 됩니다. 디스크 속도는 굉장히 느리기 때문에 **page cache** 와 비슷한 느낌으로 자주 사용되는 데이터를 메모리에 유지한다고 보면 될 것 같습니다. 파일에는 **메타데이터(Metadata)** 라 불리는 파일의 속성을 저장하는 필드가 있습니다. 파일의 메타데이터에는 다음과 같은 값들이 있습니다. <br>

![lec1_2](./images/Lec1_2.png) <br>

기타 메타데이터로는 Time, date, user id (uid), Directory 정보 등이 있습니다. 그리고 리눅스 커널에서는 **inode** 라는 자료구조에서 이러한 메타데이터를 저장합니다. 그리고 inode는 디스크에 저장되어 `Persistence`를 제공합니다. 이런 메타데이터도 **caching** 을 하는데, 파일에 접근할 때마다 디스크에서 메타데이터를 읽고, 그 위치 정보를 바탕으로 다시 디스크에 접근하면 성능이 저하됩니다. 이를 해결하기 위해 `TLB`와 유사하게 메타데이터를 메모리에 **caching** 합니다. 이러한 캐싱은 **데이터 불일치(inconsistency)** 문제를 야기할 수 있는데, 이를 극복하기 위해 `jornal`에 임시 저장하는 **저널링 파일 시스템** 이 등장하였습니다. 또한 파일이 삭제될 때는 실제 데이터가 아니라 메타데이터가 먼저 삭제되기 때문에 포렌식 및 복구가 가능합니다. <br>

![lec1_3](./images/Lec1_3.png) <br>

파일 시스템이 올라간다.. 라는 것의 의미는 파티션에 메타데이터가 설치되고 뒤에 데이터 블락(파일 블락)들이 따라오는 것이라고 볼 수 있습니다. 이 파티션들은 물리적으로 하나의 저장매체일 수 있고, 나아가서 여러 스토리지들을 묶어서 하나의 파티션으로 정의할 수 있습니다. 메타데이터가 망가지면 전체를 읽지 못할 수 있기 때문에, 복사를 중간 중간 해두는 경우도 있습니다.

### File Operations(Generic)
어떤 파일 시스템을 쓰더라도 다음 동작들은 반드시 있어야 합니다. <br>

![lec1_4](./images/Lec1_4.png) <br>

리눅스가 독특한 점은 위치가 **implicit** 하다는 점입니다. write 같은 시스템 콜을 할 때 **file descriptor** 를 사용하는데, 위치를 인자로 갖지는 않습니다. 즉 파일의 현재 위치를 프로그래밍 하는 사람이 알고 있어야 한다는 것입니다. 

- current-file-position 포인터: 파일을 읽거나 쓸 때마다 자동으로 업데이트 되는 포인터로, 파일에 대해 프로세스가 어디까지 작업을 했는지 기록하는 포인터입니다. 이는 **per-process** open-file table에 저장됩니다. 
- lseek: read, read, write 같은 호출을 반복하면 파일 포인터가 계속 뒤로 가는데, 이럴 때 **file seek** 을 이용해서 원하는 위치로 가야합니다. 

### File 사용 순서
open과 close는 여러 프로세스가 안전하게 파일을 공유할 수 있도록 지원하는 중요한 역할을 하며, 이를 위해 커널은 두 가지 테이블을 관리합니다. => **open-file table** <br>

![lec1_5](./images/Lec1_5.png) <br>

1. Per-process table
프로세스마다 독립적으로 유지되며, 파일 포인터 등의 **state** 를 저장합니다.

2. System wide
모든 프로세스가 공유하며, 파일 위치, 접근 날짜, 열린 횟수(open count) 등 프로세스에 독립적인 정보를 가집니다. 이 테이블은 디스크의 메타데이터(파일 제어 블록, file control block)를 캐싱하는 역할을 합니다. `open` 시스템 콜은 open count를 1 증가시키고, `close` 시스템 콜은 1 감소시킵니다. 즉, open을 하고 close를 해주지 않으면 이 open count가 0이 되지 않아 삭제 작업을 수행할 수 없게 됩니다. 

## Directory Structure
디렉터리는 파일들을 그룹화하여 관리하는 기법입니다.
- 단일 레벨(single-level) 디렉터리: 초창기 구조로, 모든 파일이 하나의 디렉터리에 존재하였고 하부 디렉터리를 만들 수 없었습니다.
- 트리 구조(tree-structure) 디렉터리: 현대적인 파일 시스템 구조로, 디렉터리 안에 또 다른 sub-directory를 임의로 만들 수 있습니다. 이를 위해서는 **root, path, current directory** 와 같은 개념이 필요합니다. 또한 **asyclic graph** 를 유지해야 합니다.

### Link
링크는 하나의 파일이나 디렉터리를 여러 경로에서 접근할 수 있도록 하는 기능입니다. <br>

![lec1_6](./images/Lec1_6.png) <br>

- Soft Link, Symbolic Link: 원본 파일을 가리키는 **경로 정보(symbolic name)** 를 담고 있는 특별한 파일입니다. 바로가기 같은 느낌이고, 링크 파일을 삭제해도 원본 파일은 영향을 받지 않습니다. 소프트링크가 가리키고 있던 원본 파일이 삭제되면 그 링크는 **broken link** 가 됩니다.
- Hard Link: 하나의 파일 데이터(inode)에 여러 개의 이름을 부여하는 느낌으로, 링크를 삭제하면 참조 카운트를 감소시키고 이 카운트가 0이 될 때 실제 파일 데이터가 삭제됩니다. inode에는 이 파일에 연결된 이름(하드 링크)이 몇 개인지 세는 `link count`가 있습니다. 사용자가 파일을 삭제했을 때 만약 카운트가 0이 아니라면, 즉 다른 하드 링크가 남아있다면 inode와 데이터는 디스크에 그대로 남겨둡니다. 카운트가 0이 될 때 비로소 해당 inode와 연결된 데이터 블록을 디스크에서 완전히 삭제하는 것입니다.

![lec1_7](./images/Lec1_7.png) <br>

## Mount
마운트는 특정 디렉터리에 다른 장치(device)의 파일 시스템을 연결하는 기능입니다. 파일 시스템을 사용하려면 마운트를 해야 하는데, 비어 있는 `/usr` 디렉터리에 disk2 장치를 마운트하면, disk2의 파일들이 /usr 하위 경로에서 접근 가능해지는 방식입니다.

다음과 같은 장점들이 있습니다.
- 확장성(Scalability): 수백 개의 디스크를 하나의 논리적인 파일 시스템처럼 사용할 수 있습니다.
- 투명성(Transparency): 장치가 변경되거나 추가되어도 사용자는 동일한 경로로 접근하여 사용할 수 있습니다.
- 분산 파일 시스템(Distributed FS)으로 확장 용이: 네트워크 상의 다른 컴퓨터 파일 시스템(ex: NFS)을 로컬 디렉터리에 마운트하여 내 파일처럼 쓸 수 있습니다. 

**Namespace:** 이름을 붙여 대상을 식별하는 공간을 의미합니다. 파일 시스템에서는 파일이나 디렉터리를 서로 겹치지 않게 구분하기 위해 이 개념을 사용합니다. **/a** 와 **/b** 는 서로 다른 디렉터리, 즉 **서로 다른 네임스페이스** 이므로 같은 `cc`라는 이름을 갖는 파일이 `/a/cc`, `/b/cc` 형태로 존재할 수 있습니다. 

일반적으로 시스템에 연결된 모든 저장장치가 마운트 되면 시스템에서 실행되는 모든 프로세스는 **동일한 파일 시스템 구조(네임 스페이스)** 를 보게 됩니다. 그러나, 프로세스의 파일 **file name space** 를 다르게 할 수도 있습니다. **Container** 기술을 사용하면 운영체제는 특정 프로세스에게 실제 시스템의 파일 구조와는 다른, **격리된 가상의 파일 네임 스페이스** 를 할당할 수 있습니다. 컨테이너 안에서 실행되는 프로세스는 컨테이너 내부의 파일 시스템을 root directory로 인식하게 됩니다. 

## Protection, Access Control
파일에 대한 부적절한 접근을 막는 기능입니다. RWX로 나뉩니다. 접근 권한을 부여하기 위해 사용자를 `세 그룹`으로 나눕니다.

- Owner: 파일을 소유한 사용자
- Group: 파일이 속한 그룹의 사용자들(/etc/group에 정의됩니다)
- Public/Others: 그 외 모든 사용자 

Access Control은 **chmod** 로, Owner와 Group은 **chown** 으로 변경할 수 있습니다.

761 => 소유자는 rwx, 그룹은 rw, 다른 사용자는 x(1)

# Lecture 2
## File 디자인
강의에서 다루고 있는 파일 시스템은 **커널에서 사용하는 소프트웨어** 를 의미하는 것입니다. 파일 디자인의 핵심은 **data block** 이 어디 있느냐에 대한 것으로 file의 내용을 담고 있는 **data block** 이 저장되는 디스크상 위치 정보를 결정하는 것과 이에 대한 과리 방법을 주로 다루게 될 것입니다.

Data block이 저장되는 위치의 할당 기법에는 다음 항목들이 있습니다.
- Contiguous Allocation(연속 할당)
- Linked List Allocation(연결 리스트 할당)
- Linked List Allocation Using Index(인덱스를 사용한 연결 리스트 할당)
- Inode

### Contiguous Allocation
파일을 물리적으로 연속된 disk block에 저장하는 방식입니다. 프로그래밍에서 기억 공간을 할당할 때 **고정 길이 배열** 을 사용하는 것와 유사합니다. 디렉터리는 각 파일의 **시작 위치** 와 **블록 길이** 만 기억하고 있으면 됩니다. 구현이 간단하고, 전체 파일을 한번에 읽어 들일 경우에 성능이 매우 뛰어나다는 장점이 있으나, 파일은 반드시 한번에 끝까지 기록되어야 하고, 확장을 위하여 파일의 끝에 예비용 block을 남겨두는 경우 공간 낭비가 발생한다는 단점이 있습니다. <br>

![lec1_8](./images/Lec1_8.png) <br>

위 그림의 경우, 블록이 삭제될 때 반드시 파일들은 연속적으로 할당되어야 하여 `Massive Copy`가 발생하는 것을 볼 수 있습니다. <br>

![lec_9](./images/Lec1_9.png) <br>

디스크에서 **빈 공간(disk free block)** 을 찾을 때는 **Bitmap** 을 이용합니다. 블락이 사용 중이면 0으로 표시하는 방식입니다. 컴퓨터는 비트맵을 워드 단위로 처리를 하는데, **#zero word** 는 모든 비트가 사용 중인 워드를 의미합니다. 그렇기에 (Number of bits per word) * `#zero word` + `offset of first 1`으로 free block을 찾습니다. 

### Linked List Allocation
Disk의 block을 linked list로 구현하여 file의 data를 저장하도록 하는 방식입니다. File의 data block이 disk의 어디든지 위치할 수 있고, 연속 할당과 달리 공간의 낭비가 없다는 장점이 있습니다. 그러나 Random access가 불가능합니다. 즉, File의 특정 위치를 찾기 위해서는 해당 file의 시작 노드부터 찾아가야 합니다. 이를 해결하려면 File의 메타데이터와 내용을 **In-memory object** 로 구성할 수 있습니다. 그리고 다음 data block에 대한 포인터를 저장하는 공간 역시 추가로 필요합니다. 아래 그림에서 disk block pointer의 크기를 결정하는 요소는 **전체 디스크 블록의 개수, 디스크 크기와 블록 크기** 라고 할 수 있습니다. <br>

![lec_10](./images//Lec1_10.png) <br>

Linked List Allocation 방식의 예로는 MS-DOS의 **FAT(File Allocation Table)** 파일 시스템을 들 수 있습니다. 아래 그림을 설명하자면, test라는 이름이 파일의 시작 블록은 217이고 FAT에 따라 **217 -> 618 -> 339** 블록 순서로 파일이 구성되는 것입니다. <br>

![lec1_11](./images/Lec1_11.png) <br>

### Linked List Allocation Using Index
이 방식은 file의 data block의 위치를 별도의 block에 모아두는 것입니다. file의 data block 중 하나를 **index block** 이라 하여, 모든 data block의 위치를 index block에서 알 수 있게 하는 것입니다. 이전에 봤던 Linked List Allocation과 비교하여, random access시 하나의 data block에서 찾아가고자 하는 data block의 위치를 알 수 있으므로 **더 빠르게 random access** 가 가능합니다. 그러나 Index Block의 크기가 고정되어 있어 수용할 수 있는 포인터 수에 한계가 있고, 그렇기에 최대 파일의 크기가 고정이 됩니다. 

### inode
inode는 file에 대한 data block index를 계층 형태로 관리하는 방식입니다. 큰 메모리 영역을 관리할 때 1-level paging 보다 **multi-level paging** 이 유리한 것과 비슷한 개념입니다. inode의 구성 요소는 다음과 같습니다.
- File에 대한 속성 나타내는 field: mode, owners, timestamps, size, count...
- 작은 크기의 파일을 위한 direct index
- 파일의 크기가 커짐에 따라서 요구되는 data block의 index들을 저장하기 위한 index table들
    - single indirect block
    - double indirect block
    - triple indirect block
직접 블록이 아닌 간접 간접 참조 블록의 경우 inode -> 포인터 블록 -> 포인터 블록 -> data 블록들 이런 방식으로 관리를 합니다. <br>

![lec1_12](./images//Lec1_12.png) <br>

inode의 용량을 계산해보겠습니다. 일단 블록 사이즈가 4KB, 직접 블록의 개수는 12, 포인터 크기가 4 바이트라고 가정을 하겠습니다.

- Direct Blocks: 4KB * 12 => 48KB
- Single Indirect: 4KB * 1024 => 4MB
- Double Indirect: 4KB * 1024 * 1024 => 4GB
- Triple Indirect: 4KB * 1024 * 1024 * 1024 => 4TB

## Directory 구현
**Directory Entry**: directory를 표현하기 위한 자료구조를 의미합니다. 파일 시스템에 따라서 directory entry를 구성하는 field도 달라집니다. 일반적인 directory entry는 파일의 이름, 속성과 같은 정보가 저장되지만 inode를 사용하는 경우에는 file name과 inode number만 저장이 됩니다.

### Directory in MS-DOS
![lec1_13](./images/Lec1_13.png) <br>

File의 구현이 linked list allocation인 MS-DOS에서 사용하는 directory entry는 위 그림과 같습니다. **first block number** 를 통해 맨 처음 data block만 알아내면 전체 file의 data를 알 수 있게 됩니다.

### Directory in Linux
![lec1_14](./images/Lec1_14.png) <br>
MS-DOS와 다르게 file의 속성, ownership 같은 정보는 해당하는 file의 inode 자료구조에 들어있기 때문에 directory entry에는 `file 이름`과 `inode number`가 있습니다. 이제 리눅스에서 directory lookup을 하는 과정을 알아보겠습니다. 예시로 들 경로는 **/usr/ast/mbox** 파일 입니다. <br>

![lec1_15](./images//Lec1_15.png) <br>

먼저 루트 디렉토리에서 `usr` 디렉터리의 inode 번호인 6을 알아냅니다. 그리고 inode 6을 확인하면 direct blocks 필드의 132 값을 통해 /usr 디렉터리를 나타내고 있는 것이 132 블록임을 알아낼 수 있습니다. 이 과정을 반복하여 inode 60이 가리키고 있는 실제 데이터 블록을 확인하면 파일 내용을 읽을 수 있게 됩니다. 그러나 위 과정에서 알 수 있듯 상당히 많은 disk I/O가 발생하기 때문에 **directory cache** 를 통해 디스크 접근 횟수를 대폭 감소시킬 수 있습니다. 자주 접근하는 경로의 inode와 directory entry를 메모리에 캐싱하는 것입니다. 

# Lecture 3
## File System의 계층화
![lec3_1](./images/Lec3_1.png) <br>

파일 시스템을 계층화 하면 코드 재사용성 극대화할 수 있습니다. **I/O control** 계측은 모든 파일 시스템에서 공통으로 사용하기에 파일 시스템 코드의 중복을 최소화 할 수 있습니다. 이제 각 계층을 알아보겠습니다.

1. Logical file system
파일 시스템 메타 데이터를 관리합니다(파일 이름, 크기, 권한, timestamp, inode 등)

2. File-organization
**논리 블록 주소를 물리 블록 주소로 변환** 하는 계층입니다. 논리 블록은 파일 내에서 0부터 N까지 순차적을 번호를 매긴 블록을 의미하고, 물리 블록은 실제 저장장치의 물리 주소를 의미합니다. 사용자가 파일의 100번째 블록을 요청하면 디스크의 5432번 블록 주소로 바꿔주는 이런 느낌입니다.

3. Basic file system
장치 드라이버에게 저장장치의 **물리 블록을 읽고 쓰도록 명령** 을 내리는 계층입니다. File-organization에서 5432번을 읽으라는 명령을 주면 장치 드라이버에게 5432번 블록을 읽으라는 명령을 내립니다. 

4. I/O Control
**하드웨어와 직접 통신** 하는 계층입니다. 장치 드라이버가 저장장치 하드웨어에 맞게 명령어를 전달합니다. 예를 들어, Basic file system에서 5432번 블록을 읽으라는 명령을 주면 I/O control 계층에서 하드웨어에 맞는 명령어로 변환을 하고, Device는 실제 디스크 헤드를 이동하여 이를 읽습니다. 

## VFS(Virtual File System)
VFS는 다양항 논리 파일 시스템을 추상화하고, 여러 파일 시스템에 대해 **동일한 system call interface(API)** 를 제공합니다. 이러한 API를 통해 VFS는 여러 파일 시스템을 uniform하게 사용하고, 여러 파일 시스템이 있어도 1개만 있는 것처럼 프로그래밍을 할 수 있습니다. 즉 **object-oriented** 방식을 사용하는 것입니다. 
```
// VFS 구조체
struct file_operations {
    int (*read)(file, buffer, size);   // 함수 포인터
    int (*write)(file, buffer, size);  // 함수 포인터
};

// ext4 구현
struct file_operations ext4_ops = {
    .read = ext4_read,
    .write = ext4_write
};

// NTFS 구현
struct file_operations ntfs_ops = {
    .read = ntfs_read,
    .write = ntfs_write
};

// VFS는 이렇게 호출합니다
file->ops->read(file, buffer, size); 
```

## 파일 시스템의 자료구조
![lec3_2](./images/Lec3_2.png) <br>

파일 시스템에는 두 가지 자료구조가 있습니다: **On-disk**, **In memory**

### On-disk
디스크에 영구 저장되는 자료구조입니다. 각 요소를 알아봅시다.
1. Boot block
디스크의 첫 번째 block으로 운영체제가 시작하기 위해 필요한 정보를 가지고 있습니다(부트로더 코드, 커널 위치, 초기화 정보 등..)
2. Super block
파일 시스템 관련 정보들이 어디에 저장되어 있는지에 대한 메타데이터입니다. 파일 시스템 타입, Inode 테이블 위치, 전체 블록 개수, 남은 블록 개수 등이 있습니다. **파일 시스템 당 하나씩** 주어지고, `/` 위치를 가지고 있습니다. 손상되면 복구가 안되기 때문에 중복해서 저장하며 마운트 시에 메모리에 복사되어 메모리 버전이 사용이 됩니다. **inode table** 의 위치를 저장하고 있습니다. 
3. File Control Block (FCB)
Linux File System에서는 FCB는 **inode** 로 구현됩니다. 파일의 모든 메타데이터를 저장합니다. 
4. Directory structure
파일들을 파일 시스템 내에서 organize 하기 위한 자료구조 입니다. **inode + 파일 이름** 으로 directory structure를 구성합니다. 

![lec3_3](./images/Lec3_3.png) <br>

위 그림은 파일 시스템의 디스크 레이아웃입니다. **inode table blocks** 는 inode를 모아놓은 block을 의미하며, 여러 block을 사용해 inode들을 저장합니다. 그림을 보면 Super Block 백업도 있는 것을 확인할 수 있습니다. 파일 접근의 디스크 I/O를 계산해보면 
1. Super Block 읽기 (inode 테이블 위치 확인)
2. Inode 읽기 (파일 메타데이터)
3. Data Block 읽기 (실제 파일 내용)

못해도 3번의 disk access가 필요하다는 것을 알 수 있습니다. 성능 향상을 위해 memory에 caching을 할 수 있습니다. => **Buffer Cache** 