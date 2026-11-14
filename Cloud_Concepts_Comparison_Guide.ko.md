# 클라우드 개념 비교 가이드

> 기업이 클라우드를 사용하는 이유, 온프레미스와 퍼블릭 클라우드의 차이, AWS·Google Cloud·Microsoft Azure·Oracle Cloud Infrastructure(OCI)·VMware 환경의 공통 개념과 용어를 정리한 포트폴리오 문서입니다.

## 1. 기업은 왜 클라우드를 사용하는가?

클라우드는 기업이 서버·네트워크·스토리지 같은 IT 자원을 필요한 만큼 요청하고, 빠르게 만들고, 사용량에 따라 관리할 수 있게 합니다. NIST는 클라우드의 주요 특성으로 온디맨드 셀프서비스, 네트워크를 통한 접근, 자원 풀링, 빠른 탄력성, 측정 가능한 서비스를 제시합니다. [NIST 클라우드 컴퓨팅 정의](https://www.nist.gov/publications/nist-definition-cloud-computing)

### 기업이 얻는 주요 이점

1. **서비스 출시 속도 향상**<br>
   서버 조달, 랙 설치, 네트워크 연결을 기다리는 대신 포털·API·코드로 인프라를 만들 수 있습니다. 개발·테스트 환경을 빠르게 준비하고 필요가 끝나면 정리할 수 있습니다.

2. **수요 변화에 대한 대응**<br>
   쇼핑 행사, 월말 정산, 캠페인처럼 사용량이 급증하는 시기에 자원을 늘리고, 한산할 때 줄일 수 있습니다. 자동 확장 설정과 애플리케이션 설계가 함께 필요합니다.

3. **초기 자본 지출의 전환**<br>
   자체 데이터센터를 먼저 구축하는 대신, 소비한 클라우드 자원에 따라 비용을 지불하는 사용량 기반 모델을 활용할 수 있습니다. 다만 클라우드가 항상 더 저렴한 것은 아닙니다. 장기 상시 사용량, 데이터 전송, 라이선스, 운영 인력, 예약·약정 조건까지 함께 비교해야 합니다.

4. **글로벌 인프라 활용**<br>
   여러 지역의 리전과 가용 영역을 이용해 사용자와 가까운 곳에 서비스를 배치하거나 장애 복구 구조를 설계할 수 있습니다. 실제 가용성은 단순히 클라우드에 올렸다는 사실만으로 확보되지 않으며, 다중 영역 구성과 백업·복구 계획이 필요합니다.

5. **관리형 서비스 활용**<br>
   데이터베이스, 메시징, 분석, AI 같은 관리형 서비스를 이용하면 기업이 직접 관리해야 할 서버 운영 업무를 줄일 수 있습니다. 대신 서비스별 제약, 가격, 보안 설정, 데이터 이동 가능성을 검토해야 합니다.

6. **자동화와 표준화**<br>
   Terraform, CloudFormation, ARM/Bicep 등의 IaC(Infrastructure as Code)를 사용하면 인프라 구성을 코드로 검토·버전 관리·재현할 수 있습니다. 개발·검증·운영 환경을 같은 방식으로 배포하고 변경 이력을 관리하는 데 도움이 됩니다.

7. **비즈니스와 운영의 민첩성**<br>
   팀이 인프라 요청을 기다리는 시간을 줄이고, 작은 단위로 실험하고 개선할 수 있습니다. 이 효과를 얻으려면 비용 가시성, 권한 통제, 로그·모니터링, 배포 절차도 같이 설계해야 합니다.

### 클라우드가 자동으로 해결해 주지 않는 것

- **보안과 운영 책임이 사라지는 것은 아닙니다.** 제공자는 물리 데이터센터와 기반 인프라를 관리하지만, 고객은 데이터·계정·권한·구성·애플리케이션을 계속 관리해야 합니다. 책임 경계는 IaaS/PaaS/SaaS에 따라 달라집니다. [Azure 공유 책임 모델](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility), [Google Cloud 공유 책임과 Shared Fate](https://docs.cloud.google.com/architecture/framework/security/shared-responsibility-shared-fate)
- **비용은 자동으로 절감되지 않습니다.** 방치된 VM, 과도한 사양, 불필요한 저장 데이터, 리전 간 전송은 비용을 키울 수 있습니다.
- **가용성은 설계의 결과입니다.** 백업, 복구 리허설, 다중 영역 배포, 장애 전환을 직접 설계해야 하는 경우가 많습니다.
- **클라우드 이전만으로 현대화가 끝나지 않습니다.** 기존 서버를 그대로 가상 머신에 옮기는 방식(lift and shift)은 이전 속도를 높일 수 있지만, 운영 방식과 애플리케이션의 한계도 함께 가져올 수 있습니다.

## 2. 온프레미스와 클라우드의 차이

| 비교 항목 | 온프레미스(On-premises) | 퍼블릭 클라우드(Public cloud) |
|---|---|---|
| 장비 소유 | 기업이 서버·스토리지·네트워크 장비를 구매하고 소유 | 클라우드 제공자가 기반 장비를 소유·운영하고 고객은 서비스를 사용 |
| 시작 방식 | 조달, 설치, 네트워크·보안 구성 후 사용 | 콘솔, API 또는 코드로 자원 프로비저닝 |
| 용량 변경 | 추가 장비 구매와 설치가 필요해 시간이 걸릴 수 있음 | 제공되는 범위에서 비교적 빠르게 증설·축소 가능 |
| 비용 구조 | 초기 투자(CAPEX), 유지보수·전력·공간 비용 비중이 큼 | 사용량 기반 운영비(OPEX) 중심. 약정·예약 할인도 가능 |
| 물리 시설 관리 | 기업이 데이터센터, 전력, 냉각, 물리 보안을 책임 | 제공자가 클라우드 데이터센터와 기반 하드웨어를 관리 |
| 제어 범위 | 하드웨어와 시설까지 깊은 제어 가능 | 서비스 모델에 따라 제어 범위가 다름 |
| 유지보수 | 하드웨어 교체와 용량 계획을 기업이 수행 | 기반 하드웨어 유지보수는 제공자가 수행. OS·애플리케이션 관리는 서비스 종류에 따라 고객 책임 |
| 지역·재해 복구 | 별도 데이터센터와 회선 등 추가 투자가 필요 | 여러 리전·영역을 활용할 수 있으나, 이중화·복구 구성은 별도로 설계해야 함 |
| 연결성 | 사내망 중심 구성이 가능 | 콘솔과 API 접근, 네트워크 연결 및 ID 보호를 안전하게 구성해야 함 |
| 적합할 수 있는 상황 | 특수 하드웨어·지연시간·규제·기존 투자 등으로 직접 제어가 중요한 경우 | 빠른 프로비저닝, 가변 수요, 관리형 서비스, 글로벌 배포가 중요한 경우 |

**핵심 차이:** 클라우드도 결국 물리적인 서버와 네트워크 위에서 실행됩니다. 차이는 자원을 누가 소유하고 운영하는지, 소비자가 어느 계층까지 직접 관리하는지, 자원을 어떤 방식으로 요청·확장·과금하는지에 있습니다.

### 하이브리드 클라우드

기업은 모든 시스템을 한 번에 옮기지 않고, 일부는 온프레미스에 두고 일부는 퍼블릭 클라우드에서 운영할 수 있습니다. 이를 하이브리드 구성이라고 부릅니다. 예를 들어 규제 데이터베이스는 사내 환경에 유지하면서 웹/API 계층은 퍼블릭 클라우드에서 운영할 수 있습니다. 이때 네트워크 연결, ID 연동, 관측성, 데이터 동기화, 장애 대응을 함께 설계해야 합니다.

## 3. 용어를 읽기 전에 알아둘 점

### IaaS, PaaS, SaaS

- **IaaS:** 가상 머신, 가상 네트워크, 디스크 등 기본 인프라를 제공받습니다. OS와 그 위의 애플리케이션 운영 책임은 대체로 고객에게 있습니다.
- **PaaS:** 런타임·데이터베이스 같은 관리 플랫폼을 사용합니다. 고객은 애플리케이션과 데이터에 더 집중하고 기반 OS·서버 관리 업무를 제공자에게 더 맡깁니다.
- **SaaS:** 완성된 소프트웨어를 서비스로 사용합니다. 고객은 계정, 데이터, 권한, 설정 등을 관리합니다.

같은 회사의 클라우드 안에서도 서비스 모델에 따라 책임 경계가 달라집니다. [Azure 공유 책임 모델](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility)

### Region, Availability Zone, 데이터센터

- **Region(리전):** 클라우드 제공자가 서비스를 제공하는 지리적 영역입니다.
- **Availability Zone(가용 영역):** 한 리전 안에서 장애 영향을 분리하도록 구성된 영역 단위입니다. 벤더별 물리·논리 구현은 다를 수 있습니다.
- **데이터센터:** 실제 서버와 네트워크 장비가 설치된 시설입니다. 고객이 클라우드를 사용한다고 해서 보통 특정 데이터센터의 위치나 장비를 직접 관리하는 것은 아닙니다.
- **VMware 환경:** 조직이 운영하는 데이터센터 안의 vSphere 클러스터·워크로드 도메인이나, 클라우드 사업자가 제공하는 VMware 서비스 위에서 VM을 운영할 수 있습니다. VMware 자체를 AWS·Azure·OCI와 동일한 유형의 퍼블릭 클라우드 사업자로 단순 분류하면 안 됩니다.

## 4. AWS, Google Cloud, Azure, OCI, VMware 개념 대응표

아래 표는 기능상 가장 가까운 개념을 보여 줍니다. **같은 행은 비슷한 목적을 가진다는 뜻이지, 제품 기능·제약·과금·운영 방식이 완전히 같다는 뜻은 아닙니다.** OCI는 Oracle Cloud Infrastructure를 뜻합니다. VMware 열은 주로 vSphere/VMware Cloud Foundation(VCF) 기반 프라이빗 클라우드 또는 VMware 서비스 환경을 가리킵니다.

| 개념 | AWS | Google Cloud | Microsoft Azure | Oracle Cloud Infrastructure (OCI) | VMware (vSphere/VCF 중심) | 설명 |
|---|---|---|---|---|---|---|
| 컴퓨트 VM | Amazon EC2 instance | Compute Engine VM instance | Azure Virtual Machine | OCI Compute VM instance (Compute는 베어메탈 인스턴스도 제공) | vSphere Virtual Machine | CPU·메모리·OS를 가진 가상 서버. VMware VM과 EC2는 둘 다 VM이지만, EC2는 AWS가 제공·과금·통합 관리하는 클라우드 서비스이고 vSphere VM은 VMware 가상화 플랫폼에서 생성·운영하는 워크로드 객체입니다. |
| 이미지 / VM 원본 | AMI (Amazon Machine Image) | Machine image / boot disk image | VM image / Azure Compute Gallery image | Platform image 또는 custom image | VM template / content library template | OS와 소프트웨어 구성이 담긴 배포 원본. 이미지 형식, 복제 방식, 패치·공유 정책은 서로 다릅니다. |
| 인스턴스 사양 청사진 | Launch template | Instance template | VM scale set model / ARM·Bicep 구성 | Compute instance configuration | VM template 또는 Aria Automation Cloud Template | 새 VM의 사양과 배포 설정을 재사용합니다. AWS/GCP의 인스턴스 템플릿과 VMware의 VM 템플릿은 이름과 쓰임이 겹치지만 동일한 객체는 아닙니다. |
| 서비스 카탈로그 / Blueprint | CloudFormation template/Stack, Service Catalog 제품 | Terraform 구성·Infrastructure Manager 배포 | ARM template/Bicep 배포, Azure Deployment Stacks | Resource Manager stack/template (Terraform IaC이며 서비스 카탈로그와 직접 대응하지 않음) | VMware Cloud Foundation Automation의 Cloud Template 및 Catalog (기존 VMware Aria Automation 명칭으로도 알려짐) | Blueprint는 일반 용어로 재사용 가능한 인프라 설계도·요청 항목을 뜻합니다. AWS의 CloudFormation 템플릿, Azure Bicep, Terraform 구성, VMware Cloud Template는 구현 방식과 수명주기가 다릅니다. |
| Workflow / 오케스트레이션 | AWS Step Functions, Systems Manager Automation | Google Cloud Workflows, Cloud Composer | Azure Logic Apps, Azure Automation | 이벤트 기반 작업을 위한 Events + Functions (범용 workflow와 직접 대응하지 않음) | VCF Automation 워크플로/확장, vCenter·vRO 기반 자동화(설치 구성에 따라) | 여러 단계의 작업과 조건·재시도·연동을 정의합니다. 단, 배포 템플릿(IaC)과 실행 흐름(workflow)은 관련은 있어도 서로 다른 역할입니다. |
| 계정·조직 경계 | AWS Organization → Account | Organization → Folder → Project | Microsoft Entra tenant → Management Group → Subscription → Resource Group | Tenancy → compartments | VCF Organization/Project(사용 제품에 따라), vCenter 인벤토리의 Datacenter/Folder/Resource Pool | 리소스 소유·분리·정책·청구의 경계. 구조가 1:1로 대응하지 않습니다. 특히 Azure의 tenant와 subscription은 다른 계층입니다. VMware는 사용 중인 VCF/자동화 제품 구성에 따라 테넌트 경계가 달라질 수 있습니다. |
| 사용자·권한 관리(IAM) | AWS IAM; Identity Center | Cloud IAM (조직·폴더·프로젝트·리소스에 역할 부여) | Microsoft Entra ID + Azure RBAC | OCI IAM users, groups, tenancy/compartment 범위의 policy statement | vCenter SSO 및 역할·권한; VCF Automation의 프로젝트/역할(구성에 따라) | 누가 어떤 리소스에 어떤 작업을 할 수 있는지 제어합니다. 정책 문법, 상속 범위, 역할 설계는 다릅니다. 여기서 “계정 IAM”은 보통 계정 및 ID·접근관리(IAM)를 의미합니다. |
| 워크로드 ID | IAM role / instance profile | Service account | Managed identity / service principal | Instance principal (dynamic group + IAM policy) | vSphere/VCF 서비스 계정 또는 외부 ID 연동 | 사람이 아닌 VM·앱·자동화가 다른 서비스를 호출할 때 쓰는 ID입니다. 장기 비밀번호를 소스 코드에 넣는 대신, 역할과 단기 자격 증명 또는 관리형 ID를 쓰는 패턴이 권장됩니다. |
| 가상 네트워크 | Amazon VPC, subnet, security group | VPC network, subnet, firewall rules | Virtual Network (VNet), subnet, Network Security Group | Virtual Cloud Network (VCN), subnet, route table, security list / network security group, gateways | NSX segment, gateway, distributed firewall 또는 vSphere port group | 네트워크를 논리적으로 분리하고 연결·통신을 제어합니다. VPC와 VNet은 유사한 개념이지만 기능과 네트워크 범위가 서로 다릅니다. VMware에서는 vSphere 네트워킹만 쓰는지 NSX를 쓰는지에 따라 기능이 달라집니다. |
| VM용 블록 스토리지 | Amazon EBS volume | Persistent Disk / Hyperdisk | Azure Managed Disk | Block Volume 및 Boot Volume | VMDK on datastore; VCF 환경에서는 vSAN datastore 등이 가능 | VM의 OS 디스크나 데이터 디스크처럼 블록 장치로 붙이는 스토리지입니다. 성능 등급, 복제 범위, 스냅샷·백업 정책은 서비스별로 비교해야 합니다. |
| 객체 스토리지 | Amazon S3 | Cloud Storage | Azure Blob Storage | OCI Object Storage (Standard, Infrequent Access, Archive tier) | VCF/vSphere 자체에 범용 퍼블릭 객체 스토리지가 자동으로 포함되는 것은 아님. 별도 제품·외부 스토리지를 구성 | 파일을 객체와 키로 저장하는 서비스입니다. VM에 붙는 블록 디스크와 다른 용도입니다. VMware 환경에서 S3 호환 스토리지나 오브젝트 플랫폼을 연결할 수 있지만 이는 별도 구성입니다. |
| 파일 스토리지 | Amazon EFS / FSx | Filestore / Google Cloud NetApp Volumes | Azure Files / Azure NetApp Files | OCI File Storage (NFSv3) | NFS/SMB datastore 또는 외부 NAS·파일 서비스 | 여러 서버가 파일 공유 프로토콜로 접근하는 저장소입니다. 서비스별 프로토콜·성능·가용 범위가 다릅니다. |
| 오토스케일링 / VM 그룹 | EC2 Auto Scaling Group | Managed Instance Group | Virtual Machine Scale Sets | Compute instance pool + autoscaling configuration | vSphere/VCF 자동화 또는 외부 운영 도구로 VM 그룹·확장 구성 | 부하나 정책에 따라 VM 수를 늘리거나 줄이는 기능. VMware의 자동화는 설치된 제품과 설계에 따라 별도 구현이 필요할 수 있어, 퍼블릭 클라우드의 관리형 스케일 그룹과 완전히 같지 않습니다. |
| 로드 밸런서 | Elastic Load Balancing (ALB/NLB 등) | Cloud Load Balancing | Azure Load Balancer / Application Gateway | OCI Load Balancer; OCI Network Load Balancer (L3/L4) | NSX Advanced Load Balancer 등 별도 제품·구성 | 트래픽을 여러 백엔드로 분산합니다. 계층(L4/L7), 기능, 가격, 배포 범위를 비교해야 합니다. |
| 관리형 관계형 데이터베이스 | Amazon RDS / Aurora | Cloud SQL / AlloyDB | Azure SQL Database / Azure Database for PostgreSQL 등 | Autonomous AI Database Serverless; Base Database Service; Exadata Database Service (Dedicated Infrastructure / Exascale) | VCF의 VM에 직접 설치하거나 별도 VMware/외부 DB 서비스 사용 | VM에 데이터베이스를 직접 설치하는 것과 관리형 DB 서비스를 사용하는 것은 운영 책임이 크게 다릅니다. VMware 가상화만으로 퍼블릭 클라우드의 관리형 DB와 같은 운영 서비스가 자동 제공되는 것은 아닙니다. |

### OCI 적합성, 가격, 리전별 제공 조건

OCI는 VM·베어메탈 Compute와 관리형 Oracle 데이터베이스를 제공합니다. Autonomous AI Database Serverless는 프로비저닝, 패치, 백업, 확장 등 데이터베이스 운영을 자동화하고, Base Database Service는 VM 기반 DB 시스템에서 고객에게 더 많은 제어권을 제공합니다. Oracle DB 서비스를 필요로 하거나 VM과 전용 물리 컴퓨트 중 선택해야 하는 워크로드에 적합할 수 있지만, 서비스에 따라 운영 책임이 다릅니다. OCI의 tenancy/compartment 기반 정책, VCN 설계, Terraform 중심의 Resource Manager는 다른 클라우드 상품과 그대로 교환할 수 없는 OCI 고유 개념입니다.

OCI 가격은 선택한 리소스와 사용량에 따라 책정됩니다. 모든 구성에 하나의 가격이 적용된다고 가정하지 말고 [OCI 비용 계산기 및 최신 가격표](https://www.oracle.com/cloud/pricing/)를 확인하세요. Free Tier에는 최대 30일 동안 사용할 수 있는 미화 300달러의 체험 크레딧과 별도의 Always Free 사용량이 있으며, 자격 조건과 서비스별 한도가 적용됩니다. Always Free Compute 및 Autonomous AI Database 리소스는 테넌시의 홈 리전에서만 만들 수 있고, US Government Cloud 리전에서는 Free Tier 리소스를 사용할 수 없습니다. 무료 사용량을 전제로 하기 전에 최신 할당량, 리전별 제공 서비스, 약관을 확인하세요.

OCI 리전마다 하나 이상의 Availability Domain이 있을 수 있고, 서비스나 shape의 제공 여부도 리전별로 다를 수 있습니다. 용량·고가용성·재해 복구를 설계할 때 대상 리전과 서비스 문서를 확인하세요.

### Blueprint, 템플릿, Workflow를 구분하면

1. **Image(이미지):** VM 부팅에 쓰는 OS·디스크 원본.
2. **VM Template / Instance Template:** VM의 사양과 기본 설정을 재사용하는 틀.
3. **Infrastructure Template / Blueprint:** VM뿐 아니라 네트워크·디스크·권한 등 여러 리소스의 원하는 구성을 선언하는 설계도.
4. **Workflow:** 트리거 이후 승인, 리소스 생성, 설정, 검사, 알림 같은 작업을 순서·조건·오류 처리와 함께 실행하는 절차.

예를 들어 “Linux VM과 두 개의 디스크, 특정 서브넷, 모니터링 에이전트”를 선언하는 것은 템플릿/IaC에 가깝고, “요청 접수 → 관리자 승인 → VM 배포 → 구성 관리 실행 → 결과 알림”은 워크플로에 가깝습니다. 한 제품이 두 역할을 연결하거나 함께 제공할 수도 있습니다.

## 5. “VM = EC2”인가?

**개념 수준에서는 맞습니다.** Amazon EC2는 AWS에서 가상 서버를 제공하는 서비스이고, EC2 instance는 그 서비스에서 실행되는 VM입니다. AWS 공식 문서도 EC2 인스턴스를 AWS 클라우드의 가상 서버로 설명합니다. [Amazon EC2 사용 설명서](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)

다만 다음과 같이 말하는 편이 더 정확합니다.

- **VM**은 가상화된 컴퓨터라는 일반 개념입니다.
- **EC2**는 AWS의 컴퓨트 서비스 이름입니다.
- **OCI Compute**는 Oracle의 컴퓨트 서비스입니다. Compute 인스턴스는 VM 또는 베어메탈일 수 있으므로, VM 개념을 비교할 때는 VM 인스턴스를 기준으로 봅니다.
- **EC2 instance**는 EC2에서 생성한 VM 리소스입니다.
- **vSphere VM**은 VMware 하이퍼바이저 환경에서 관리되는 VM 리소스입니다.
- 그러므로 `VM ≈ EC2 instance ≈ Compute Engine VM ≈ Azure VM ≈ OCI Compute VM instance`는 초급 개념 비교로 유용하지만, 서비스 스펙트럼·계정 구조·네트워크·스토리지·가격·운영 기능까지 같다는 뜻은 아닙니다.

클라우드 업체는 VM을 제공하는 것 외에도 과금, 이미지, 네트워크, 보안 그룹/방화벽, 디스크, 자동 확장, 모니터링, API, SLA를 묶어 하나의 관리형 서비스를 제공합니다. 이 서비스 묶음이 플랫폼별 차이를 만듭니다. AWS EC2는 고객이 VM 내부 OS와 애플리케이션을 관리해야 하는 IaaS 예시입니다. [AWS EC2 개념](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)

## 6. 실무 예시: 웹 애플리케이션 한 대 배포

아래는 벤더별 구성 요소가 어떤 역할을 하는지 보여 주는 개념 예시입니다.

```text
사용자
  │
  ▼
DNS → Load Balancer
          │
          ▼
     VM / Compute Instance
       ├── Boot disk (block storage)
       ├── Service identity (권한 최소화)
       └── Object storage (이미지·로그 등)

구성 정의: Terraform / CloudFormation / Bicep / OCI Resource Manager / VMware Cloud Template 등
단계 자동화: CI/CD pipeline 또는 workflow engine
관측 및 운영: logs + metrics + alerts + backup + patching
```

AWS라면 EC2·EBS·S3·VPC·IAM이 대표적이고, Google Cloud라면 Compute Engine·Persistent Disk·Cloud Storage·VPC·Cloud IAM, Azure라면 Azure VM·Managed Disk·Blob Storage·VNet·Entra ID/RBAC, OCI라면 Compute·Block Volume·Object Storage·VCN·IAM이 대응되는 역할을 할 수 있습니다. VMware 기반 프라이빗 클라우드에서는 vSphere VM·VMDK/datastore·NSX·vCenter/VCF Automation 등이 구성에 따라 역할을 나눠 담당합니다. 이 목록은 개념 예시이며, 구성 요소를 그대로 바꾸어 끼울 수 있다는 보장은 없습니다.

## 7. 멀티클라우드에서 재사용되는 개념과 달라지는 부분

### 공통으로 통하는 운영 개념

- 컴퓨트, 네트워크, 스토리지, ID, 이미지, 템플릿, 정책, 로그, 모니터링, 백업, 태그/라벨, 비용관리
- 원하는 상태를 선언하고 반복 가능한 배포로 관리하는 IaC
- 최소 권한 원칙, 환경 분리(dev/test/prod), 변경 이력 관리
- 복구 목표(RPO/RTO), 가용성, 장애 도메인, 용량 계획

### 벤더마다 다시 확인할 항목

- 계층 구조와 정책·권한의 상속 범위
- 리전·존의 물리적 의미와 서비스 제공 범위
- 네트워크 주소, 라우팅, 방화벽, 사설 연결 방법
- 디스크와 객체/파일 스토리지의 성능·복제·비용 모델
- 이미지와 템플릿의 형식, 공유, 업데이트 절차
- API, IaC provider, 워크플로 엔진, 관리형 서비스의 제약
- 사용량 과금, 예약·약정, 데이터 전송 비용, 라이선스

**좋은 비교 방법:** 상품 이름을 억지로 1:1로 대응하기보다, “이 서비스가 제공하는 기능은 무엇인가?”, “내가 관리하는 계층은 어디까지인가?”, “장애·보안·과금 책임은 누가 갖는가?”를 비교합니다.

## 8. Snowflake·Databricks와 주요 클라우드 사업자 비교

### 먼저 제품 범주를 구분하기

Snowflake와 Databricks는 관리형 서비스로 제공되는 데이터 플랫폼입니다. 범용 퍼블릭 클라우드 사업자가 아니며 AWS, Microsoft Azure, Google Cloud, Oracle Cloud Infrastructure(OCI)를 직접 대체하지 않습니다. 하이퍼스케일러는 컴퓨트, 네트워크, 스토리지, ID, 데이터베이스, 분석, AI 등 광범위한 인프라·플랫폼 제품을 제공합니다. Snowflake와 Databricks는 데이터 워크로드에 집중하며 지원되는 하이퍼스케일러 인프라에서 실행됩니다.

브랜드 이름을 억지로 대응시키기보다 각 제품이 제공하는 기능과 운영 책임을 비교하세요. Snowflake 또는 Databricks를 클라우드 네이티브 스토리지, 네트워크, ID 및 분석 서비스와 함께 사용할 수도 있습니다.

### 플랫폼과 운영 모델 비교

| 항목 | Snowflake | Databricks | 하이퍼스케일러 제품 |
|---|---|---|---|
| 핵심 모델 | 공유 데이터 계층, 독립적으로 크기를 조정하는 Virtual Warehouse, Cloud Services 계층을 갖춘 관리형 데이터 플랫폼 | Apache Spark, Delta Lake, Unity Catalog를 중심으로 SQL, 데이터 엔지니어링, 스트리밍, 데이터 과학, AI 워크로드를 제공하는 Lakehouse 플랫폼 | 고객이 조합해 사용하는 인프라 서비스 및 여러 데이터·분석 제품으로 구성된 광범위한 클라우드 포트폴리오 |
| 스토리지와 컴퓨트 | Snowflake가 데이터 저장소를 관리하고 Virtual Warehouse는 별도 컴퓨트로 독립 확장됩니다. 일반 Snowflake 테이블을 사용할 때 고객이 기반 버킷이나 DB 서버를 관리하지 않습니다. | 컴퓨트는 클라우드 객체 스토리지와 분리되어 실행됩니다. Lakehouse 데이터는 Delta Lake, Apache Iceberg 같은 개방형 테이블 형식을 주로 사용합니다. 관리형 자산과 외부 자산은 저장소 수명주기 책임이 다릅니다. | 서비스마다 다릅니다. 예를 들어 BigQuery는 관리형 스토리지와 컴퓨트를 분리하고, Redshift·객체 스토리지·서버리스 분석·클러스터 서비스는 각각 모델이 다릅니다. |
| 대표 워크로드 | SQL 분석, 데이터 웨어하우스, 데이터 공유, 데이터 애플리케이션, 구조화·반구조화 데이터 및 일부 비정형 데이터의 거버넌스 접근 | 배치·스트리밍 데이터 엔지니어링, Lakehouse SQL/BI, 머신러닝, AI, Spark의 장점을 활용하는 워크로드 | 객체 스토리지와 관리형 웨어하우스부터 스트림 처리, 데이터 통합, ML, 범용 컴퓨트까지 폭넓은 워크로드 |
| 관리형 서비스 경계 | Snowflake가 플랫폼, 업그레이드, 기반 인프라를 관리합니다. Snowflake 계정은 지원되는 퍼블릭 클라우드 리전에서 호스팅되며 고객이 사설 클라우드에 직접 설치하지 않습니다. | Databricks가 서비스 Control Plane을 관리합니다. 데이터 플레인 위치와 인프라 책임은 클라우드 및 컴퓨트 유형에 따라 다르며 Serverless와 고객 구성형 컴퓨트의 책임 경계도 동일하지 않습니다. | 클라우드 사업자가 인프라를 운영합니다. 고객 책임, 구성 선택지, 관리 수준이 서로 다른 여러 서비스를 선택할 수 있습니다. |
| 생태계와 이식성 | SQL 중심 생태계, 커넥터, 내장 데이터 공유, 일부 개방형 형식 지원. Snowflake 관리 테이블은 Snowflake 최적화 저장 형식을 사용합니다. | Spark 및 오픈소스 생태계, 클라우드 객체 스토리지, Delta Lake, Apache Iceberg, 노트북, ML·데이터 엔지니어링 도구. 이식성은 사용 형식·기능·클라우드 연동에 따라 달라집니다. | 해당 사업자의 IAM·네트워크·스토리지·운영 서비스와 긴밀히 통합되지만, 아키텍처가 특정 사업자에 종속될 수 있습니다. |
| 거버넌스와 보안 | Snowflake 역할 및 권한, 객체 권한, OAuth, Horizon Catalog 기능. 기반 클라우드 IAM은 Snowflake 권한을 대체하지 않습니다. | Unity Catalog가 지원되는 데이터·AI 자산을 통합 관리하고 권한, 계보, 감사 기능을 제공합니다. 클라우드 IAM 및 저장소 구성도 여전히 중요합니다. | 클라우드 IAM과 서비스별 제어가 사업자 리소스를 관리합니다. 데이터 수준 권한 및 거버넌스 기능은 제품에 따라 다릅니다. |
| 배포 및 벤더 관계 | Snowflake가 지원되는 AWS, Azure, Google Cloud 인프라에서 서비스를 운영합니다. 고객은 Snowflake 계정 및 서비스 인터페이스를 사용하며, 기반 사업자의 전체 제품군을 이용하는 것은 아닙니다. | 지원되는 클라우드에서 사용 가능하며 서비스와 데이터 플레인의 동작은 클라우드, 리전, 컴퓨트 방식에 따라 달라집니다. 기반 클라우드 계정과 서비스를 대체하지 않습니다. | AWS, Azure, Google Cloud, OCI가 각자의 클라우드와 네이티브 서비스를 운영합니다. 특정 클라우드에서 실행되는 제품이 그 사업자 전체와 동등해지는 것은 아닙니다. |
| 비용 고려사항 | 컴퓨트 크레딧, Warehouse 크기와 유휴 시간, 저장소, 데이터 전송, 에디션·기능, 약정 비용을 고려합니다. Warehouse를 분리하면 워크로드별 사용량을 격리할 수 있지만 비용이 자동으로 예측 가능해지는 것은 아닙니다. | Databricks 사용료(예: DBU), 해당되는 경우 클라우드 컴퓨트·스토리지, Serverless 요율, 네트워크·데이터 전송, 약정 비용을 고려합니다. 실제 청구는 배포·컴퓨트 방식에 따라 달라집니다. | 서비스별 컴퓨트, 스토리지, 요청, 네트워크 전송, 지원, 예약·약정 요금이 적용됩니다. 목록 가격만 보지 말고 실제 아키텍처와 사용량을 비교해야 합니다. |
| 주요 장단점 | 편리한 관리형 SQL/데이터 경험과 워크로드 격리. 플랫폼 고유 기능과 과금 모델이 있고 기반 인프라 제어는 제한적입니다. | 유연한 Lakehouse, 엔지니어링·AI 워크플로, 개방형 형식 지원. 거버넌스할 아키텍처 선택지가 많고 플랫폼과 클라우드에 비용이 분산될 수 있습니다. | 선택의 폭과 기반 인프라 통합성이 높지만, 필요한 제품을 선정·통합·운영해야 합니다. |

### 클라우드 네이티브 제품 예시: 1:1 대응이 아님

| 클라우드 사업자 | 대표 데이터·분석 서비스 | 비교 시 유의점 |
|---|---|---|
| AWS | Amazon S3, AWS Glue, Amazon Athena, Amazon Redshift, Amazon EMR, Amazon SageMaker | 각 서비스는 서로 다른 계층을 담당합니다. Redshift는 관리형 데이터 웨어하우스이고 S3는 객체 스토리지입니다. Glue, Athena, EMR, SageMaker의 통합·운영 방식도 각각 다릅니다. |
| Microsoft Azure | OneLake 및 Microsoft Fabric 경험, Azure Databricks, Azure Data Lake Storage, Azure Synapse 구성 요소, Azure Machine Learning | Fabric은 통합 분석 플랫폼이고 Azure Databricks는 Azure에서 제공되는 Databricks 서비스입니다. 둘 다 Azure 전체와 같은 뜻이 아닙니다. 제품 기능과 경계는 계속 바뀔 수 있습니다. |
| Google Cloud | BigQuery, Cloud Storage, Dataflow, Dataproc, Vertex AI | BigQuery는 스토리지·컴퓨트를 분리한 관리형 분석 플랫폼입니다. 나머지 서비스는 스토리지·처리·AI 기능을 각각 제공합니다. |
| OCI | Object Storage, Autonomous AI Database, Oracle Database 서비스, OCI Data Flow, OCI Data Science | OCI는 광범위한 클라우드 제품군 안에서 관리형 Oracle DB 및 데이터 처리 선택지를 제공합니다. OCI 전체가 Snowflake 또는 Databricks의 단순한 “OCI 버전”은 아닙니다. 제품별 현재 리전 및 서비스 제공 여부를 확인하세요. |

예를 들어 BigQuery와 Snowflake는 스토리지와 컴퓨트를 분리한 관리형 분석을 제공하지만 아키텍처, 거버넌스, 생태계, 과금, 클라우드와의 관계가 다릅니다. Amazon Redshift는 AWS 전체 플랫폼과 동등한 제품이 아니라 클라우드 네이티브 웨어하우스입니다. Databricks는 하이퍼스케일러에서 실행되고 해당 객체 스토리지를 사용할 수 있지만, 기반 클라우드 사업자가 아닌 데이터·AI 플랫폼입니다.

### 어떤 제품을 선택할까?

- **Snowflake 선택:** 관리형 SQL 중심 분석, 워크로드별로 격리된 Warehouse, 거버넌스된 데이터 공유, 낮은 인프라 운영 부담이 중요할 때 적합합니다.
- **Databricks 선택:** Lakehouse 접근, Spark 기반 엔지니어링·스트리밍, 개방형 테이블 형식, 분석·ML·AI 워크플로 통합이 핵심일 때 적합합니다.
- **클라우드 네이티브 서비스 선택:** 기존 클라우드 환경과의 긴밀한 통합, 특정 네이티브 서비스, 또는 더 넓은 인프라 제어가 중요할 때 적합합니다. 단일 통합 제품이 아니라 여러 보완 제품으로 구성될 수 있습니다.
- **복수 플랫폼 함께 사용:** 사용 사례가 필요성을 정당화한다면 가능하지만 ID, 거버넌스, 데이터 이동, 관측성, 기술 역량, 플랫폼 간 비용을 계획해야 합니다. 분명한 이유 없이 플랫폼 간 데이터를 복제하지 마세요.

결정 전에 대표 쿼리와 파이프라인을 시험하고, 데이터가 실제로 저장되는 위치, ID·거버넌스 책임, 리전별 제공 여부, 유휴·피크 사용량과 데이터 전송을 포함한 비용, 종료·상호운용성 요구사항을 확인하세요.

## 9. 한눈에 보는 정리

- 기업은 빠른 구축, 탄력적 확장, 관리형 서비스, 글로벌 배포, 자동화를 위해 클라우드를 사용합니다.
- 온프레미스와 클라우드는 물리 인프라의 소유·운영 주체, 자원 조달 방식, 제어 범위가 다릅니다.
- `VM`은 일반 개념이고 `EC2`는 AWS 서비스입니다. EC2 instance는 AWS가 제공하는 VM입니다.
- AWS·Google Cloud·Azure·OCI는 공통된 인프라 개념을 제공하지만 제품의 기능과 계층 구조는 동일하지 않습니다.
- OCI는 VM·베어메탈 Compute와 관리형 Oracle DB 선택지가 필요한 워크로드에 맞을 수 있습니다. 실제 구성의 비용과 리전별 서비스·shape 한도를 확인해야 합니다.
- VMware는 vSphere/VCF 기반 가상화·프라이빗 클라우드 플랫폼입니다. 퍼블릭 클라우드와 비교할 때는 플랫폼 전체가 아니라 VM, 네트워크, 스토리지, 자동화 등 각 기능 계층을 기준으로 비교하는 편이 정확합니다.
- IAM, VM, 스토리지, 템플릿, 워크플로는 모든 환경에서 중요하지만, 용어가 비슷하다는 이유만으로 정책·동작·과금이 같다고 가정하면 안 됩니다.
- Snowflake와 Databricks는 관리형 데이터 플랫폼이며 AWS·Azure·Google Cloud·OCI와 직접 동등하지 않습니다. 특정 사업자의 네이티브 서비스와 워크로드, 제어 경계, 거버넌스, 총비용을 기준으로 비교하세요.

## 참고 자료

- [NIST — The NIST Definition of Cloud Computing](https://www.nist.gov/publications/nist-definition-cloud-computing)
- [Microsoft Learn — Shared responsibility in the cloud](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility)
- [Google Cloud — Shared responsibilities and shared fate](https://docs.cloud.google.com/architecture/framework/security/shared-responsibility-shared-fate)
- [AWS — What is Amazon EC2? / EC2 concepts](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)
- [Snowflake — Key concepts and architecture](https://docs.snowflake.com/en/user-guide/intro-key-concepts)
- [Databricks — What is a data lakehouse?](https://docs.databricks.com/aws/en/lakehouse)
- [Databricks — What is Unity Catalog?](https://docs.databricks.com/aws/en/data-governance/unity-catalog)
- [AWS — What is Amazon Redshift?](https://docs.aws.amazon.com/redshift/latest/mgmt/welcome.html)
- [Microsoft Learn — What is Microsoft Fabric?](https://learn.microsoft.com/en-us/fabric/fundamentals/microsoft-fabric-overview)
- [Google Cloud — BigQuery overview](https://docs.cloud.google.com/bigquery/docs/introduction)
- [Google Cloud — Infrastructure Manager overview](https://docs.cloud.google.com/infrastructure-manager/docs/overview)
- [Google Cloud — Grant a workflow permission to access resources](https://docs.cloud.google.com/workflows/docs/authentication)
- [Google Cloud — Compute Engine IAM roles and permissions](https://docs.cloud.google.com/compute/docs/access/iam)
- [Microsoft Learn — Azure Resource Manager templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-create-first-template)
- [Microsoft Learn — Connect Azure Blob Storage from Logic Apps workflows](https://learn.microsoft.com/en-us/azure/connectors/connectors-create-api-azureblobstorage)
- [Oracle — Compute service overview](https://docs.oracle.com/en-us/iaas/Content/Compute/Concepts/computeoverview.htm)
- [Oracle — Networking and VCN overview](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/overview.htm)
- [Oracle — Identity and Access Management overview](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/overview.htm)
- [Oracle — Calling services from an instance using instance principals](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm)
- [Oracle — Resource Manager overview](https://docs.oracle.com/en-us/iaas/Content/ResourceManager/Concepts/resourcemanager.htm)
- [Oracle — Block Volume overview](https://docs.oracle.com/en-us/iaas/Content/Block/Concepts/overview.htm)
- [Oracle — Object Storage overview](https://docs.oracle.com/en-us/iaas/Content/Object/Concepts/objectstorageoverview.htm)
- [Oracle — File Storage overview](https://docs.oracle.com/en-us/iaas/Content/File/Concepts/filestorageoverview.htm)
- [Oracle — Creating instance pools](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/creatinginstancepool.htm)
- [Oracle — Load Balancer overview](https://docs.oracle.com/en-us/iaas/Content/Balance/Concepts/balanceoverview.htm)
- [Oracle — Network Load Balancer overview](https://docs.oracle.com/en-us/iaas/Content/NetworkLoadBalancer/overview.htm)
- [Oracle — Events overview](https://docs.oracle.com/en-us/iaas/Content/Events/Concepts/eventsoverview.htm)
- [Oracle — Functions overview](https://docs.oracle.com/en-us/iaas/Content/Functions/Concepts/functionsoverview.htm)
- [Oracle — OCI Database service overview](https://docs.oracle.com/en-us/iaas/Content/Database/Concepts/databaseoverview.htm)
- [Oracle — Autonomous AI Database Serverless documentation](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/index.html)
- [Oracle — Regions and Availability Domains](https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm)
- [Oracle — OCI Free Tier terms and limits](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier.htm)
- [Oracle — OCI pricing and cost estimator](https://www.oracle.com/cloud/pricing/)
- [Google Cloud — VMware Cloud Foundation components and networking](https://docs.cloud.google.com/vmware-engine/docs/vmware-solution/concepts-vcf)
- [VMware — VCF technical validation](https://www.vmware.com/docs/vmware-esg-technical-validation-cloud-foundation)

---

*문서 작성일: 2026-11-12. 벤더 제품과 명칭은 변경될 수 있으므로 설계·구현 전 공식 문서에서 현재 제공 범위와 구성 조건을 확인하세요.*
