---
lab:
    title: '06 - 트래픽 관리 구현'
    module: '모듈 06 - 네트워크 트래픽 관리'
---

# 랩 06 - 트래픽 관리 구현
# 학생 랩 매뉴얼

## 랩 시나리오

허브 및 스포크 네트워크 토폴로지의 Azure 가상 머신을 대상으로 하는 네트워크 트래픽 관리를 테스트하는 작업을 맡았습니다. 또한 Contoso는 (이전 랩에서 테스트한 메시 토폴로지를 만드는 대신에) Azure 환경에서 이러한 구현을 고려하고 있습니다. 이 테스트에서는 허브를 통해 트래픽의 흐름을 만들어내는 사용자 정의 경로에 의존하여 스포크 간의 연결을 구현하고 계층 4 및 계층 7의 부하 분산 장치를 사용하여 가상 머신 전반의 트래픽 분포를 구현해야 합니다. 이를 위해 Azure Load Balancer(계층 4) 및 Azure Application Gateway(계층 7)를 사용하고자 합니다.

>**참고**: 이 랩은 기본적으로 배포를 위해 선택한 지역의 Standard_Dsv3 시리즈에서 사용할 수 있는 총 8개의 vCPU가 필요한데, 이는 4개의 Standard_D2s_v3 SKU Azure VM 배포가 포함되기 때문입니다. 학생이 4개의 vCPU 한도가 있는 평가판 계정을 사용하는 경우, vCPU가 하나만 필요한 VM 크기(예: Standard_B1s)를 사용할 수 있습니다. 

## 목적

이 랩에서는 다음 작업을 수행합니다.

+ 작업 1: 랩 환경 프로비전
+ 작업 2: 허브 및 스포크 네트워크 토폴로지 구성
+ 작업 3: 가상 네트워크 피어링의 전이성 테스트
+ 작업 4: 허브 및 스포크 토폴로지의 라우팅 구성
+ 작업 5: Azure Load Balancer 구현
+ 작업 6: Azure Application Gateway 구현

## 예상 시간: 60분

## 지침

### 연습 1

#### 작업 1: 랩 환경 프로비전

이 작업에서는 동일한 Azure 지역에 4개의 가상 머신을 배포합니다. 처음 두 개는 허브 가상 네트워크에 있고 나머지 각각은 별도의 스포크 가상 네트워크에 있습니다.

1. [Azure Portal](https://portal.azure.com)에 로그인합니다.

1. Azure Portal에서 오른쪽 상단의 아이콘을 클릭하여 **Azure Cloud Shell**을 엽니다.

1. **Bash**또는 **PowerShell** 중 하나를 선택하라는 메시지가 표시되면 **PowerShell**을 선택합니다.      

    >**참고**: **Cloud Shell**을 처음 시작할 때 **탑재된 스토리지가 없음** 메시지가 나타나면 이 랩에서 사용 중인 구독을 선택하고 **스토리지 만들기**를 클릭합니다. 

1. Cloud Shell 창의 도구 모음에서 **파일 업로드/다운로드** 아이콘을 클릭합니다. 드롭다운 메뉴에서 **업로드**를 클릭하고 파일 **\\Allfiles\\Module_06\\az104-06-vms-template.json**, **\\Allfiles\\Labs\\06\\az104-06-vm-template.json**, **\\Allfiles\\Labs\\06\\az104-06-vm-parameters.json**을 Cloud Shell 홈 디렉터리로 업로드합니다.

1. Cloud Shell 창에서 다음을 실행하여 첫 번째 가상 네트워크와 가상 머신 쌍을 호스팅할 첫 번째 리소스 그룹을 만듭니다(`[Azure_region]` 자리 표시자를 Azure 가상 머신을 배포하고자 하는 Azure 지역의 이름으로 바꿉니다.).

   ```pwsh
   $location = '[Azure_region]'

   $rgName = 'az104-06-rg1'

   New-AzResourceGroup -Name $rgName -Location $location
   ```
1. Cloud Shell 창에서 다음을 실행하여 첫 번째 가상 네트워크를 만들고 업로드한 템플릿 및 매개 변수 파일을 사용하여 가상 머신 쌍을 배포합니다.

   ```pwsh
   New-AzResourceGroupDeployment `
      -ResourceGroupName $rgName `
      -TemplateFile $HOME/az104-06-vms-template.json `
      -TemplateParameterFile $HOME/az104-06-vm-parameters.json `
      -AsJob
   ```

1. Cloud Shell 창에서 다음을 실행하여 두 번째 가상 네트워크 및 세 번째 가상 머신을 호스트하는 두 번째 리소스 그룹을 만듭니다.

   ```pwsh
   $rgName = 'az104-06-rg2'

   New-AzResourceGroup -Name $rgName -Location $location
   ```
1. Cloud Shell 창에서 다음을 실행하여 두 번째 가상 네트워크를 만들고 업로드한 템플릿 및 매개 변수 파일을 사용하여 가상 머신을 배포합니다.

   ```pwsh
   New-AzResourceGroupDeployment `
      -ResourceGroupName $rgName `
      -TemplateFile $HOME/az104-06-vm-template.json `
      -TemplateParameterFile $HOME/az104-06-vm-parameters.json `
      -nameSuffix 2 `
      -AsJob
   ```
1. Cloud Shell 창에서 다음을 실행하여 세 번째 가상 네트워크 및 네 번째 가상 컴퓨터를 호스트하는 세 번째 리소스 그룹을 만듭니다.

   ```pwsh
   $rgName = 'az104-06-rg3'

   New-AzResourceGroup -Name $rgName -Location $location
   ```
1. Cloud Shell 창에서 세 번째 가상 네트워크를 만들고 업로드한 템플릿 및 매개 변수 파일을 사용하여 가상 머신을 배포합니다.

   ```pwsh
   New-AzResourceGroupDeployment `
      -ResourceGroupName $rgName `
      -TemplateFile $HOME/az104-06-vm-template.json `
      -TemplateParameterFile $HOME/az104-06-vm-parameters.json `
      -nameSuffix 3 `
      -AsJob
   ```
    >**참고**: 다음 작업을 진행하기 전에 배포가 완료될 때까지 기다립니다. 이 작업에 약 5분이 소요됩니다.

    >**참고**: 배포 상태를 확인하기 위해 이 작업에서 만든 리소스 그룹의 속성을 검사할 수 있습니다.

1. Cloud Shell 창을 닫습니다.

#### 작업 2: 허브 및 스포크 네트워크 토폴로지 구성

이 작업에서는 허브 및 스포크 네트워크 토폴로지를 만들기 위해 이전 작업에 배포한 가상 네트워크 간의 로컬 피어링을 구성합니다.

1. Azure Portal에서 **가상 네트워크**를 검색하고 선택합니다.

1. 이전 작업에서 만든 가상 네트워크를 검토합니다. 

    >**참고**: 세 개의 가상 네트워크 배포에 사용된 템플릿은 가상 네트워크 세 개의 IP 주소 범위가 겹치지 않도록 합니다.

1. 가상 네트워크 목록에서 **az104-06-vnet01**을 클릭합니다.

1. **설정** 섹션의 **az104-06-vnet01** 가상 네트워크 블레이드에서 **피어링**을 클릭한 다음 **+추가**를 클릭합니다.

1. 다음 설정으로 피어링을 추가합니다(다른 설정은 기본값을 그대로 둡니다.)

    | 설정 | 값 |
    | --- | --- |
    | az104-06-vnet01에서 원격 가상 네트워크로 피어링 이름 | **az104-06-vnet01_to_az104-06-vnet2** |
    | 가상 네트워크 배포 모델 | **리소스 관리자** |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 가상 네트워크 | **az104-06-vnet2 (az104-06-rg2)** |
    | az104-06-vnet2에서 az104-06-vnet01로 피어링 이름 | **az104-06-vnet2_to_az104-06-vnet01** |
    | az104-06-vnet01에서 az104-06-vnet2로 가상 네트워크 액세스 허용 | **사용** |
    | az104-06-vnet2에서 az104-06-vnet01로 가상 네트워크 액세스 허용 | **사용** |
    | az104-06-vnet2에서 az104-06-vnet01로 전달된 트래픽 허용 | **사용 안 함** |
    | az104-06-vnet01에서 az104-06-vnet2로 전달되는 트래픽 허용 | **사용** |
    | 게이트웨이 전송 허용 | **(확인란 선택 취소)** |

    >**참고**: 작업이 완료될 때까지 기다립니다.

    >**참고**: 이 단계에서는 하나는 az104-06-vnet01에서 az104-06-vnet2로, 다른 하나는 az104-06-vnet2에서 az104-06-vnet01로 두 개의 로컬 피어링을 설정합니다.

    >**참고**: 이 랩의 후반부에서 구현할 스포크 가상 네트워크 간의 라우팅을 쉽게 하기 위해 **전달된 트래픽 허용**을 사용하도록 설정해야 합니다.

1. **설정** 섹션의 **az104-06-vnet01** 가상 네트워크 블레이드에서 **피어링**을 클릭한 다음 **+추가**를 클릭합니다.

1. 다음 설정으로 피어링을 추가합니다(다른 설정은 기본값을 그대로 둡니다.)

    | 설정 | 값 |
    | --- | --- |
    | az104-06-vnet01에서 원격 가상 네트워크로 피어링 이름 | **az104-06-vnet01_to_az104-06-vnet3** |
    | 가상 네트워크 배포 모델 | **리소스 관리자** |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 가상 네트워크 | **az104-06-vnet3 (az104-06-rg3)** |
    | az104-06-vnet3에서 az104-06-vnet01로 피어링 이름 | **az104-06-vnet3_to_az104-06-vnet01** |
    | az104-06-vnet01에서 az104-06-vnet3로 가상 네트워크 액세스 허용 | **사용** |
    | az104-06-vnet3에서 az104-06-vnet01로 가상 네트워크 액세스 허용 | **사용** |
    | az104-06-vnet3에서 az104-06-vnet01로 전달된 트래픽 허용 | **사용 안 함** |
    | az104-06-vnet01에서 az104-06-vnet3로 전달되는 트래픽 허용 | **사용** |
    | 게이트웨이 전송 허용 | **(확인란 선택 취소)** |

    >**참고**: 이 단계에서는 하나는 az104-06-vnet01에서 az104-06-vnet3로, 다른 하나는 az104-06-vnet3에서 az104-06-vnet01로 두 개의 로컬 피어링을 설정합니다. 이렇게 하면 허브 및 스포크 토폴로지 설정(두 개의 스포크 가상 네트워크 사용)이 완료됩니다.

    >**참고**: 이 랩의 후반부에서 구현할 스포크 가상 네트워크 간의 라우팅을 쉽게 하기 위해 **전달된 트래픽 허용**을 사용하도록 설정해야 합니다.

#### 작업 3: 가상 네트워크 피어링의 전이성 테스트

이 작업에서는 Network Watcher를 사용하여 가상 네트워크 피어링의 전이성을 테스트합니다.

1. Azure Portal에서 **Network Watcher**를 검색하고 선택합니다.

1. **Network Watcher** 블레이드에서 Azure 지역 목록을 펼치고 이 랩의 첫 번째 작업에 리소스를 배포한 Azure에서 서비스가 활성화되어 있는지 확인합니다.

1. **Network Watcher** 블레이드에서 **연결 문제 해결**로 이동합니다.

1. **Network Watcher-연결 문제 해결** 블레이드에서 다음 설정으로 확인을 시작합니다(다른 부분을 기본값으로 남겨둠).

    | 설정 | 값 |
    | --- | --- |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 리소스 그룹 | **az104-06-rg1** |
    | 원본 유형 | **가상 머신** |
    | 가상 머신 | **az104-06-vm0** |
    | 대상 | **수동으로 지정** |
    | URI, FQDN 또는 IPv4: | **10.62.0.4** |
    | 프로토콜 | **TCP** |
    | 대상 포트 | **3389** |

    > **참고**: **10.62.0.4는**는 **az104-06-vm2의** 개인 IP 주소를 나타냅니다.

1.  **확인**을 클릭하고 연결 확인 결과가 반환될 때까지 기다립니다. **연결 가능** 상태인지 확인합니다. 네트워크 경로를 검토하고 VM 간 중간 홉이 없는 직접적 연결이라는 점에 유의하세요.

    > **참고**: 이는 예상된 일입니다. 허브 가상 네트워크가 첫 번째 스포크 가상 네트워크로 직접 피어링하기 때문입니다.

    > **참고**: **az104-06-vm0**에 Network Watcher 에이전트 가상 머신 확장을 설치해야 하기 때문에 초기 검사는 약 2분 정도 걸릴 수 있습니다.

1. **Network Watcher-연결 문제 해결** 블레이드에서 다음 설정으로 확인을 시작합니다(다른 부분을 기본값으로 남겨둠).

    | 설정 | 값 |
    | --- | --- |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 리소스 그룹 | **az104-06-rg1** |
    | 원본 유형 | **가상 머신** |
    | 가상 머신 | **az104-06-vm0** |
    | 대상 | **수동으로 지정** |
    | URI, FQDN 또는 IPv4: | **10.63.0.4** |
    | 프로토콜 | **TCP** |
    | 대상 포트 | **3389** |

    > **참고**: **10.63.0.4**는 **az104-06-vm3**의 개인 IP 주소를 나타냅니다.

1.  **확인**을 클릭하고 연결 확인 결과가 반환될 때까지 기다립니다. **연결 가능** 상태인지 확인합니다. 네트워크 경로를 검토하고 VM 간 중간 홉이 없는 직접적 연결이라는 점에 유의하세요.

    > **참고**: 이는 예상된 일입니다. 허브 가상 네트워크는 두 번째 스포크 가상 네트워크로 직접 피어링하기 때문입니다.

1. **Network Watcher-연결 문제 해결** 블레이드에서 다음 설정으로 확인을 시작합니다(다른 부분을 기본값으로 남겨둠).

    | 설정 | 값 |
    | --- | --- |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 리소스 그룹 | **“az104:-06:rg2"** |
    | 원본 유형 | **가상 머신** |
    | 가상 머신 | **az104-06-vm2** |
    | 대상 | **수동으로 지정** |
    | URI, FQDN 또는 IPv4: | **10.63.0.4** |
    | 프로토콜 | **TCP** |
    | 대상 포트 | **3389** |

1.  **확인**을 클릭하고 연결 확인 결과가 반환될 때까지 기다립니다. **연결할 수 없음** 상태임을 유의하세요. 

    > **참고**: 이는 예상된 일입니다. 두 스포크 가상 네트워크가 서로 피어링되지 않기 때문입니다. 가상 네트워크 피어링은 비전이적입니다.

#### 작업 4: 허브 및 스포크 토폴로지의 라우팅 구성

이 작업에서는 **az104-06-vm0** 가상 머신의 네트워크 인터페이스에서 IP 전달을 활성화하여 두 개의 스포크 가상 네트워크 간 라우팅을 구성하고 테스트합니다. 운영 체제 내 라우팅을 활성화하고 스포크 가상 네트워크에서 사용자 정의 경로를 구성합니다.

1. Azure Portal에서 **가상 머신**을 검색하고 선택합니다.

1. **가상 머신** 블레이드의 가상 머신 목록에서 **az104-06-vm0**을 클릭합니다.

1. **설정** 섹션의 **az104-06-vm0** 가상 머신 블레이드에서 **네트워킹**을 클릭합니다.

1. **네트워크 인터페이스** 레이블 옆에 있는 **az104-06-nic0** 링크를 클릭한 다음 **설정** 섹션의 **az104-06-nic0** 네트워크 인터페이스 블레이드에서 **IP 구성**을 클릭합니다. 

1. **IP 전달**을 **활성화** 설정하고 변경 사항을 저장합니다. 

   > **참고**: 이 설정은 **az104-06-vm0**이 두 스포크 가상 네트워크 간의 트래픽을 라우팅하는 라우터로 작동하기 위해 필요합니다.

   > **참고**: 이제 라우팅을 지원하도록 **az104-06-vm0** 가상 머신의 운영 체제를 구성해야 합니다.

1. Azure Portal에서 **az104-06-vm0** Azure 가상 머신 블레이드로 이동하여 **개요**를 클릭합니다.

1. **작업** 섹션 **az104-06-vm0** 블레이드에서 **실행 명령**을 클릭하고 명령 목록에서 **RunPowerShellScript**를 클릭합니다.

1. **명령 스크립트 실행** 블레이드에서 다음을 입력하고 **실행**을 클릭하여 원격 액세스 Windows Server 역할을 설치합니다.

   ```pwsh
   Install-WindowsFeature RemoteAccess -IncludeManagementTools
   ```

   > **참고**: 명령이 성공적으로 완료되었음이 확인되기를 기다립니다.

1. **스크립트 명령 실행** 블레이드에서 다음을 입력하고 **실행**을 클릭하여 라우팅 역할 서비스를 설치합니다.

   ```pwsh
   Install-WindowsFeature -Name Routing -IncludeManagementTools -IncludeAllSubFeature

   Install-WindowsFeature -Name "RSAT-RemoteAccess-Powershell"

   Install-RemoteAccess -VpnType RoutingOnly

   Get-NetAdapter | Set-NetIPInterface -Forwarding Enabled
   ```

   > **참고**: 명령이 성공적으로 완료되었음이 확인되기를 기다립니다.

   > **참고**: 이제 스포크 가상 네트워크에서 사용자 정의 경로를 만들고 구성해야 합니다.

1. Azure Portal에서 **경로 테이블**을 검색하여 선택하고 **경로 테이블** 블레이드에서 **+추가**를 클릭합니다.

1. 다음 설정으로 경로 테이블을 만듭니다(다른 설정은 기본값으로 남깁니다).

    | 설정 | 값 |
    | --- | --- |
    | 이름 | **az104-06-rt23** |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 리소스 그룹 | **“az104:-06:rg2"** |
    | 위치 | 가상 네트워크를 만든 Azure 지역의 이름 |
    | 가상 네트워크 게이트웨이 경로 전파 | **사용 안 함** |

   > **참고**: 라우팅 테이블이 만들어질 때까지 기다립니다. 약 3분이 소요됩니다.

1. **경로 테이블** 블레이드로 돌아가서 **새로 고침**을 클릭한 다음 **az104-06-rt23**를 클릭합니다.

1. **az104-06-rt23** 경로 테이블 블레이드에서 **경로**를 클릭한 다음 **+ 추가**를 클릭합니다.

1. 다음 설정으로 새 경로를 추가합니다(다른 설정은 기본값으로 남겨둡니다).

    | 설정 | 값 |
    | --- | --- |
    | 경로 이름 | **az104-06-route-vnet2-to-vnet3** |
    | 주소 접두사 | **10.63.0.0/20** |
    | 다음 홉 유형 | **가상 어플라이언스** |
    | 다음 홉 주소 | **10.60.0.4** |

1. **az104-06-rt23** 경로 테이블 블레이드로 돌아가서 **서브넷**을 클릭한 다음 **+ 연결**을 클릭합니다.

1. 경로 테이블 **az104-06-rt23**을 다음 서브넷과 연결합니다.

    | 설정 | 값 |
    | --- | --- |
    | 가상 네트워크 | **az104-06-vnet2** |
    | 서브넷 | **subnet0** |

1. **경로 테이블** 블레이드로 돌아가서 **+ 추가** 를 클릭합니다.

1. 다음 설정으로 경로 테이블을 만듭니다(다른 설정은 기본값으로 남깁니다).

    | 설정 | 값 |
    | --- | --- |
    | 이름 | **az104-06-rt32** |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 리소스 그룹 | **az104-06-rg3** |
    | 위치 | 가상 네트워크를 만든 Azure 지역의 이름 |
    | 가상 네트워크 게이트웨이 경로 전파 | **사용 안 함** |

   > **참고**: 라우팅 테이블이 만들어질 때까지 기다립니다. 약 3분이 소요됩니다.

1. **경로 테이블** 블레이드로 돌아가서 **새로 고침**을 클릭한 다음 **az104-06-rt32**를 클릭합니다.

1. **az104-06-rt32** 경로 테이블 블레이드에서 **경로**를 클릭한 다음 **+추가**를 클릭합니다.

1. 다음 설정으로 새 경로를 추가합니다(다른 설정은 기본값으로 남겨둡니다).

    | 설정 | 값 |
    | --- | --- |
    | 경로 이름 | **az104-06-route-vnet3-to-vnet2** |
    | 주소 접두사 | **10.62.0.0/20** |
    | 다음 홉 유형 | **가상 어플라이언스** |
    | 다음 홉 주소 | **10.60.0.4** |

1. **az104-06-rt32** 경로 테이블 블레이드로 돌아가서 **서브넷**을 클릭한 다음 **+ 연결**을 클릭합니다.

1. 경로 테이블 **az104-06-rt32**을 다음 서브넷과 연결합니다.

    | 설정 | 값 |
    | --- | --- |
    | 가상 네트워크 | **az104-06-vnet3** |
    | 서브넷 | **subnet0** |

1. Azure Portal에서 **Network Watcher-연결 문제 해결** 블레이드로 이동합니다.

1. **Network Watcher-연결 문제 해결** 블레이드에서 다음 설정으로 확인을 시작합니다(다른 부분을 기본값으로 남겨둠).

    | 설정 | 값 |
    | --- | --- |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 리소스 그룹 | **“az104:-06:rg2"** |
    | 원본 유형 | **가상 머신** |
    | 가상 머신 | **az104-06-vm2** |
    | 대상 | **수동으로 지정** |
    | URI, FQDN 또는 IPv4: | **10.63.0.4** |
    | 프로토콜 | **TCP** |
    | 대상 포트 | **3389** |

1. **확인**을 클릭하고 연결 확인 결과가 반환될 때까지 기다립니다. 상태가 **연결 가능**인지 확인합니다. 네트워크 경로를 검토합니다. 그리고 트래픽이 **10.60.0.4**를 통해 라우팅되고 **az104-06-nic0** 네트워크 어댑터로 할당되는지 확인합니다. 상태가 **연결할 수 없음**일 경우 az104-06-vm0를 재시작해야 합니다.


    > **참고**: 스포크 가상 네트워크 간의 트래픽이 라우터로 작동하는 허브 가상 네트워크에 있는 가상 머신을 통해 라우팅되므로 이는 예상한 것입니다. 

    > **참고**: **Network Watcher**를 사용하여 네트워크의 토폴로지를 볼 수 있습니다.

#### 작업 5: Azure Load Balancer 구현

이 작업에서는 허브 가상 네트워크의 두 Azure 가상 머신 앞에 Azure Load Balancer를 구현합니다.

1. Azure Portal에서 **부하 분산 장치**를 검색하여 선택한 다음 **부하 분산 장치** 블레이드에서 **+ 추가**를 클릭합니다.

1. 다음 설정으로 부하 분산 장치를 만듭니다(다른 설정은 기본값으로 남깁니다).

    | 설정 | 값 |
    | --- | --- |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 리소스 그룹 | 새 리소스 그룹 **az104-06-rg4**의 이름 |
    | 이름 | **az104-06-lb4** |
    | 지역| 이 랩에서 모든 다른 리소스를 배포한 Azure 지역의 이름 |
    | 유형 | **공용** |
    | SKU | **표준** |
    | 공용 IP 주소 | **새로 만들기** |
    | 공용 IP 주소 이름 | **az104-06-pip4** |
    | 가용성 영역 | **영역 중복** |
    | 공용 IPv6 주소 추가 | **아니요** |

    > **참고**: Azure 부하 분산 장치가 프로비전될 때까지 기다립니다. 약 2분이 소요됩니다. 

1. 배포 블레이드에서 **리소스로 이동**을 클릭합니다.

1. **az104-06-lb4** 부하 분산 장치 블레이드에서 **백 엔드 풀**을 클릭하고 **+ 추가**를 클릭합니다.

1. 다음 설정으로 백 엔드 풀을 추가합니다(다른 설정은 기본값으로 남깁니다).

    | 설정 | 값 |
    | --- | --- |
    | 이름 | **az104-06-lb4-be1** |
    | 가상 네트워크 | **az104-06-vnet01** |
    | IP 버전 | **IPv4** |
    | 가상 머신 | **az104-06-vm0** | 
    | 가상 머신 IP 주소 | **ipconfig1 (10.60.0.4)** |
    | 가상 머신 | **az104-06-vm1** |
    | 가상 머신 IP 주소 | **ipconfig1 (10.60.1.4)** |

1. 백 엔드 풀이 생성되면 **상태 프로브**를 클릭한 다음 **+추가**를 클릭합니다.

1. 다음 설정으로 상태 프로브를 추가합니다(다른 설정은 기본값으로 남깁니다).

    | 설정 | 값 |
    | --- | --- |
    | 이름 | **az104-06-lb4-hp1** |
    | 프로토콜 | **TCP** |
    | 포트 | **80** |
    | 간격 | **5** |
    | 비정상 임계값 | **2** |

1. 상태 프로브가 생성될 때까지 기다린 다음, **부하 분산 규칙**을 클릭하고 **+ 추가**를 클릭합니다.

1. 다음 설정으로 부하 분산 규칙을 추가합니다(다른 설정은 기본값으로 남겨둡니다).

    | 설정 | 값 |
    | --- | --- |
    | 이름 | **az104-06-lb4-lbrule1** |
    | IP 버전 | **IPv4** |
    | 프로토콜 | **TCP** |
    | 포트 | **80** |
    | 백 엔드 포트 | **80** |
    | 백 엔드 풀 | **az104-06-lb4-be1** |
    | 상태 프로브 | **az104-06-lb4-hp1** |
    | 세션 지속성 | **없음** |
    | 유휴 제한 시간(분) | **4** |
    | TCP 재설정 | **사용 안 함** |
    | 부동 IP(Direct Server Return) | **사용 안 함** |
    | 암시적 아웃바운드 규칙 만들기 | **예** |

1. 부하 분산 규칙이 생성될 때까지 기다린 다음, **개요**를 클릭하고 **공용 IP 주소**의 값을 기록합니다.

1. 다른 브라우저 창을 시작하고 이전 단계에서 확인한 IP 주소를 탐색합니다.

1. 브라우저 창에 **az104-06-vm0의 Hello World** 또는 **az104-06-vm1의 Hello World** 메시지가 표시되는지 확인합니다.

1. 다른 브라우저 창을 열어서 이번에는 InPrivate 모드를 사용하여 대상 VM이 (메시지에 표시된 대로) 변경되는지 확인합니다. 

    > **참고**: 브라우저 창을 새로 고치거나 InPrivate 모드를 사용하여 다시 열어야 할 수 있습니다.

#### 작업 6: Azure Application Gateway 구현

이 작업에서는 스포크 가상 네트워크의 두 Azure 가상 머신 앞에 Azure Application Gateway를 구현합니다.

1. Azure Portal에서 **가상 네트워크**를 검색하여 선택합니다.

1. **가상 네트워크** 블레이드의 가상 네트워크 목록에서 **az104-06-vnet01**을 클릭합니다.

1. **az104-06-vnet01** 가상 네트워크 블레이드의 **설정** 섹션에서 **서브넷**과 **+ 서브넷**을 차례로 클릭합니다.

1. 다음 설정으로 서브넷을 추가합니다(다른 설정은 기본값으로 남깁니다).

    | 설정 | 값 |
    | --- | --- |
    | 이름 | **subnet-appgw** |
    | 주소 범위(CIDR 블록) | **10.60.3.224/27** |
    | 네트워크 보안 그룹 | **없음** |
    | 경로 테이블 | **없음** |

    > **참고**: 이 서브넷은 Azure Application Gateway 인스턴스에서 사용되며 이 작업의 나중에 배포할 것입니다. Application Gateway에는 27 또는 그 이상 크기의 전용 서브넷이 필요합니다.

1. Azure Portal에서 **애플리케이션 게이트웨이**를 검색하여 선택하고 **애플리케이션 게이트웨이** 블레이드에서 **+추가**를 클릭합니다.

1. **애플리케이션 게이트웨이 만들기** 블레이드의 **기본** 탭에서 다음 설정을 지정합니다(다른 설정은 기본값으로 남깁니다).

    | 설정 | 값 |
    | --- | --- |
    | 구독 | 이 랩에서 사용 중인 Azure 구독의 이름 |
    | 리소스 그룹 | 새 리소스 그룹 **az104-06-rg5**의 이름 |
    | 애플리케이션 게이트웨이 이름 | **az104-06-appgw5** |
    | 영역 | 이 랩에서 모든 다른 리소스를 배포한 Azure 영역의 이름 |
    | 계층 | **표준 V2** |
    | 자동 크기 조정 사용 | **아니요** |
    | 배율 단위 | **1** |
    | 가용성 영역 | **1, 2, 3** |
    | HTTP2 | **사용 안 함** |
    | 가상 네트워크 | **az104-06-vnet01** |
    | 서브넷 | **subnet-appgw** |

1. **다음: 프런트 엔드>** 을 클릭합니다. 그리고 **애플리케이션 게이트웨이 만들기** 블레이드의 **프런트 엔드** 탭에서 다음 설정을 지정합니다(다른 설정은 기본값으로 남깁니다).

    | 설정 | 값 |
    | --- | --- |
    | 프론트 엔드 IP 주소 유형 | **공용** |
    | 방화벽 공용 IP 주소| 신규 공용 IP 주소 **az104-06-pip5**의 이름 |

1. **다음: 백 엔드>** 를 클릭하고, **애플리케이션 게이트웨이 만들기** 블레이드의 **백 엔드** 탭에서 **백 엔드 풀 추가**를 클릭합니다. 그리고 **백 엔드 풀 추가** 블레이드에서 다음 설정을 지정합니다(나머지 항목은 기본값으로 둡니다).

    | 설정 | 값 |
    | --- | --- |
    | 이름 | **az104-06-appgw5-be1** |
    | 대상 없이 백 엔드 풀 추가 | **아니요** |
    | 대상 유형 | **IP 주소 또는 FQDN** |
    | 대상 | **10.62.0.4** |
    | 대상 유형 | **IP 주소 또는 FQDN** |
    | 대상 | **10.63.0.4** |

    > **참고**: 대상은 스포크 가상 네트워크 **az104-06-vm2** 및 **az104-06-vm3**에서 가상 머신의 개인 IP 주소를 나타냅니다.

1. **추가**와 **다음: 구성 >** 을 차례로 클릭하고 **애플리케이션 게이트웨이 만들기** 블레이드의 **구성** 탭에서 **+ 회람 규칙 추가**를 클릭합니다.  

1. **라우팅 규칙 추가** 블레이드의 **수신기** 탭에서 다음 설정을 지정합니다(다른 설정은 기본값으로 남겨둡니다).

    | 설정 | 값 |
    | --- | --- |
    | 규칙 이름 | **az104-06-appgw5-rl1** |
    | 수신기 이름 | **az104-06-appgw5-rl1l1** |
    | 프런트 엔드 IP | **공용** |
    | 프로토콜 | **HTTP** |
    | 포트 | **80** |
    | 수신기 유형 | **기본** |
    | 오류 페이지 URL | **아니요** |

1. **라우팅 규칙 추가** 블레이드의 **백 엔드 대상** 탭으로 전환하고 다음 설정을 지정합니다(다른 설정은 기본값으로 남겨둡니다).

    | 설정 | 값 |
    | --- | --- |
    | 대상 유형 | **백 엔드 풀** |
    | 백 엔드 대상 | **az104-06-appgw5-be1** |

1. **회람 규칙 추가** 블레이드의 **백 엔드 대상** 탭에서 **HTTP 설정** 텍스트 상자 옆의 **새 항목 추가**를 클릭하고 **HTTP 설정 추가** 블레이드에서 다음 설정을 지정합니다(다른 설정 기본값으로 유지).

    | 설정 | 값 |
    | --- | --- |
    | HTTP 설정 이름 | **az104-06-appgw5-http1** |
    | 백 엔드 프로토콜 | **HTTP** |
    | 백 엔드 포트 | **80** |
    | 쿠키 기반 선호도 | **사용 중지** |
    | 연결 드레이닝 | **사용 중지** |
    | 요청 시간 제한(초) | **20** |

1. **HTTP 설정 추가** 블레이드에서 **추가**를 클릭하세요. **회람 규칙 추가** 블레이드로 돌아가서 **추가**를 클릭하세요.

1. **다음: 태그 >** 을 클릭합니다. 그리고 **다음: 검토+만들기>** 를 누른 후에 **만들기**를 클릭합니다.

    > **참고**: Application Gateway 인스턴스가 만들어질 때까지 기다리세요. 8분 정도 걸릴 수 있습니다.

1. Azure Portal에서 **애플리케이션 게이트웨이**를 선택하세요. **애플리케이션 게이트웨이** 블레이드에서 **az104-06-appgw5**를 클릭합니다.

1. **az104-06-appgw5** 애플리케이션 게이트웨이 블레이드에서 **프런트 엔드 공용 IP 주소**의 값을 메모하세요.

1. 다른 브라우저 창을 시작하고 이전 단계에서 확인한 IP 주소를 탐색합니다.

1. 브라우저 창에 **az104-06-vm2에서 Hello World** 혹은 **az104-06-vm3에서 Hello World** 메시지가 표시되는지 확인합니다.

1. 다른 브라우저 창을 열어 이번에는 InPrivate 모드를 사용하여 대상 VM이 변경되는지 확인합니다(웹 페이지에 표시된 메시지에 기반함). 

    > **참고**: 브라우저 창을 새로 고치거나 InPrivate 모드를 사용하여 다시 열어야 할 수 있습니다.

    > **참고**: 여러 가상 네트워크에서 가상 머신을 대상으로 하는 것은 일반적인 구성은 아닙니다. 그러나 Application Gateway가 동일한 가상 네트워크의 가상 머신에서 부하를 분산하는 Azure Load Balancer와는 달리 여러 가상 네트워크(및 다른 Azure 지역 혹은 심지어 Azure 외의 엔드포인트)에서 가상 머신을 대상으로 할 수 있다는 점을 설명하기 위한 것입니다.

#### 리소스 정리

   >**참고**: 더 이상 사용하지 않는 새로 만든 모든 Azure 리소스를 제거해야 합니다. 사용하지 않는 리소스를 제거하면 해당 비용이 발생하지 않습니다.

1. Azure Portal에서 **Cloud Shell** 창 내부의 **PowerShell** 세션을 엽니다.

1. 다음 명령을 실행하여 본 모듈의 랩 전체에서 만든 모든 리소스 그룹을 나열합니다.

   ```pwsh
   Get-AzResourceGroup -Name 'az104-06*'
   ```

1. 다음 명령을 실행하여 본 모듈의 랩 전체에서 만든 모든 리소스 그룹을 삭제합니다.

   ```pwsh
   Get-AzResourceGroup -Name 'az104-06*' | Remove-AzResourceGroup -Force -AsJob
   ```

    >**참고**: 명령은 비동기적으로 실행(-AsJob 매개 변수에 의해 결정)되므로 동일한 PowerShell 세션 내에서 즉시 다른 PowerShell 명령을 실행할 수 있지만 리소스 그룹이 실제로 삭제되기까지 몇 분 정도 걸릴 수 있습니다.

#### 리뷰

이 랩에서는 다음 작업을 했습니다.

- 랩 환경 프로비전 
- 허브 및 스포크 네트워크 토폴로지 구성
- 가상 네트워크 피어링의 전이성 테스트
+ 작업 4: 허브 및 스포크 토폴로지의 라우팅 구성
+ 작업 5: Azure Load Balancer 구현
+ 작업 6: Azure Application Gateway 구현
