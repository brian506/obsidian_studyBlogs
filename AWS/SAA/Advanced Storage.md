

### 1. AWS Snow Family ⭐⭐

오프라인 물리 장치로 대용량 데이터를 AWS로 마이그레이션하거나 인터넷 없는 환경에서 엣지 컴퓨팅에 활용.

**기준**: 네트워크로 전송 시 **1주일 이상** 걸리면 Snowball 사용.

|장치|용량|특징|
|---|---|---|
|**Snowcone**|8TB HDD|초소형 2.1kg, 공간 제한 환경|
|**Snowcone SSD**|14TB SSD|Snowcone의 SSD 버전|
|**Snowball Edge Storage Optimized**|**80TB HDD**|대용량 데이터 이동, 클러스터링|
|**Snowball Edge Compute Optimized**|42TB HDD / 28TB NVMe|GPU 옵션, 머신러닝|
|**Snowmobile**|**100PB/대**|엑사바이트 규모, **10PB 이상** 시 Snowball보다 적합|

**엣지 컴퓨팅 스펙 차이:**

- Snowcone: CPU 2개, RAM 4GB
- Snowball Compute Optimized: vCPU 104개, RAM 416GiB, 선택적 GPU
- Snowball Storage Optimized: vCPU 40개, RAM 80GB, 80TB

모든 Snow 장치에서 **EC2, Lambda** 실행 가능 (IoT Greengrass 사용). 장기 배포 시 1년/3년 할인 옵션.

**AWS OpsHub**: Snow 장치를 GUI로 관리하는 소프트웨어. 이전에는 CLI만 가능했음.

> ⭐ **핵심 포인트**: **Snowball → Glacier 직접 전송 불가!** Snowball → S3 → 수명 주기 정책 → Glacier 순서로만 가능.

---

### 2. Amazon FSx ⭐⭐

AWS에서 3rd party 고성능 파일 시스템을 완전 관리형으로 제공.

|종류|프로토콜|핵심 키워드|
|---|---|---|
|**FSx for Windows**|SMB, NTFS|Windows 환경, Active Directory, DFS, **매일 S3 백업**, Multi-AZ|
|**FSx for Lustre**|-|**HPC, 머신러닝**, S3와 원활한 통합, 밀리초 미만 지연|
|**FSx for NetApp ONTAP**|NFS, SMB, **iSCSI**|다양한 OS 호환, 오토스케일링, 중복 제거, 즉각 복제|
|**FSx for OpenZFS**|NFS (v3~v4.2)|ZFS 워크로드 이전, 0.5ms 미만 지연, 100만 IOPS, 중복제거 없음|

**FSx for Lustre 배포 옵션 — 시험 빈출:**

![](../../images/스크린샷%202026-04-12%2020.34.38.png)
---

### 3. AWS Storage Gateway ⭐⭐⭐

온프레미스 ↔ AWS 클라우드를 연결하는 **하이브리드 스토리지** 서비스. 재해 복구, 백업, 계층화 스토리지, 지연 시간 감소 목적.

|게이트웨이|프로토콜|동작 방식|용도|
|---|---|---|---|
|**S3 File Gateway**|NFS, SMB|최근 사용 데이터만 로컬 캐시, 나머지는 S3|S3를 온프레미스 파일 시스템처럼 사용|
|**FSx File Gateway**|SMB, NTFS|FSx for Windows에 로컬 캐시로 접근|Windows 파일 공유, 홈 디렉토리|
|**Volume Gateway**|**iSCSI**|S3 백업 기반 블록 스토리지. Cached(최근 데이터 로컬) / Stored(전체 로컬+주기적 S3 백업)|온프레미스 블록 스토리지 백업|
|**Tape Gateway**|iSCSI|가상 테이프 라이브러리(VTL). S3/Glacier에 백업|기존 테이프 백업 시스템 대체|

**Hardware Appliance**: 온프레미스에 가상화 서버가 없는 경우 amazon.com에서 물리 어플라이언스를 구매해 설치.

---

### 4. AWS Transfer Family ⭐

S3 또는 EFS에 **FTP 계열 프로토콜**로 파일 전송하는 완전 관리형 서비스. 지원 프로토콜: FTP, FTPS, SFTP. Multi-AZ, 고가용성. Active Directory, LDAP, Okta, Cognito 등과 인증 통합 가능.

> **시험 키워드**: "FTP/SFTP로 S3에 접근해야 한다" → Transfer Family

---

### 5. AWS DataSync ⭐⭐

대용량 데이터를 **일정 기반으로 자동 동기화**하는 서비스.

- **온프레미스 → AWS**: DataSync **에이전트 필요** (NFS, SMB, HDFS, S3 API 지원)
- **AWS → AWS** (서비스 간): **에이전트 불필요**
- 동기화 대상: S3 (모든 클래스 포함 Glacier까지), EFS, FSx(Windows/Lustre/NetApp/OpenZFS)
- 복제는 **지속적이지 않고 일정에 따라** 실행 (매시간/매일/매주)
- **파일 권한 및 메타데이터 보존** (NFS POSIX, SMB 등)
- 에이전트 하나당 최대 **10Gbps**, 대역폭 제한 설정 가능

---

### 6. 스토리지 서비스 최종 비교표 ⭐⭐⭐

|서비스|특징|키워드|
|---|---|---|
|**S3**|객체 스토리지|범용, 대부분 AWS와 통합|
|**S3 Glacier**|아카이브|장기 보관|
|**EBS**|블록, 단일 EC2 연결|1인스턴스 전용|
|**Instance Store**|EC2 물리 직접 연결|최고 IOPS, 임시|
|**EFS**|네트워크 파일 시스템|Linux 전용, 다중 AZ, POSIX|
|**FSx for Windows**|SMB/NTFS|Windows + AD 환경|
|**FSx for Lustre**|HPC 병렬 분산 파일 시스템|ML, HPC, S3 통합|
|**FSx for NetApp ONTAP**|다중 OS 호환|NFS/SMB/iSCSI, 중복제거|
|**FSx for OpenZFS**|ZFS 기반|NFS, 초저지연|
|**Storage Gateway**|온프레미스↔AWS 연결|하이브리드 클라우드|
|**Transfer Family**|FTP/FTPS/SFTP|S3/EFS에 FTP 접근|
|**DataSync**|예약 데이터 동기화|온프레미스→AWS, 메타데이터 보존|
|**Snow Family**|물리 장치|오프라인 대용량 마이그레이션|

---

### 시험 TIP 최종 정리

- **Snowball → Glacier 직접 불가** → S3 경유 + 수명 주기 정책 필수
- **10PB 이상** → Snowmobile / 그 이하 → Snowball
- **HPC, ML** → FSx for Lustre / **Windows, AD** → FSx for Windows
- **iSCSI 블록 스토리지** → Volume Gateway
- **기존 테이프 백업 대체** → Tape Gateway
- **FTP로 S3 접근** → Transfer Family
- **온프레미스→AWS 정기 동기화 + 메타데이터 보존** → DataSync
- **DataSync**: 온프레미스에서 사용 시 에이전트 필요, AWS 서비스 간은 불필요