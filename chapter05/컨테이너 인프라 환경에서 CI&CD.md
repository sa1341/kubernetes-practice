# 컨테이너 인프라 환경에서 CI/CD

컨테이너 인프라 환경에서는 주로 CD를 강조하지만, CI와 CD는 대부분 함께 사용되기 때문에 우선 `CI/CD`의 개념을 정확히 이해해야 합니다.

일반적으로 CI는 코드를 커밋하고 빌드했을 때 정상적으로 작동하는지 반복적으로 검증해 애플리케이션의 신뢰성을 높이는 작업입니다.

CD는 CI 과정에서 생성된 신뢰할 수 있는 애플리케이션을 실제 상용 환경에 자동으로 배포하는 것을 의미합니다. 

## 1. 젠킨스로 쿠버네티스 운영 환경 개선하기

애플리케이션 배포 영역에 쿠버네티스를 사용하면 개발자는 애플리케이션 개발에만 집중 할 수 있게 됩니다. 

기존에는 환경이 다른 곳에서 빌드한 애플리케이션을 배포하게 되면 개발자가 개발 환경에 맞춰 애플리케이션 코드를 일일이 수정해야 했습니다. 

하지만 모든 배포환경을 컨테이너 인프라로 일원화하고, CI/CD 도구를 사용하면 애플리케이션에 맞는 환경을 적용해 자동으로 배포할 수 있습니다. 그리고 통합과정에서 만들어진 컨테이너 이미지를 기반으로 쿠버네티스가 존재하는 어떤 환경에서도 일관성 있는 애플리케이션을 배포 할 수 있습니다.

## 2. 헬름으로 젠킨스 설치

```bash
~/_Book_k8sinfra/ch5/5.3.1/jenkin-install.sh
```

젠킨스를 설치하고 마스터 노드의 상세정보를 확인하면 다음과 같습니다.

```bash
kubectl get node m-k8s -o yaml | nl
```

![image](https://user-images.githubusercontent.com/22395934/235084989-1aa0f718-e76d-4c62-9e6d-a9e85defe4da.png)

여기서 테인트라는 용어가 등장하는데 테인트는 특별하게 취급돼야 하는 곳에 설정하여, 쉽게 접근하지 못하는 중요한 것으로 설정합니다. 

즉, 현재 상태에서는 마스터 노드에 테인트가 설정돼 있어 특별한 목적으로 사용되는 노드라는 것을 명시해 두었습니다.

이번엔 디플로이먼트인 젠킨스의 상태를 살펴보면 톨러레이션이 등장 합니다. 톨러레이션의 키 값이 위에 설명한 마스터 노드의 테인트 키 값과 동일합니다. 이것이 의미하는 바는 톨러레이션이라는 특별한 키를 가져야만 이곳에 출입이 가능하다는 의미입니다. 

보통 테인트와 톨러레이션은 위와 같이 혼합해서 사용합니다.

![image](https://user-images.githubusercontent.com/22395934/235085860-b7ea9df7-76b5-4b74-ab25-6aa3b10fd511.png)

이 설정을 통해서 알 수 있는 사실은 젠킨스 컨트롤러가 여러 곳에 스케줄되지 않고 마스터 노드인 `m-k8s`에서만 스케줄 될 수 있도록 구성된 사실 입니다. 

## 3. 젠킨스 살펴보기

젠킨스의 구조를 살펴보면 젠킨스 컨트롤러는 마스터 노드에 설치하였습니다. 젠킨스 에이전트는 필요 시에 생성되고 작업을 마치면 삭제되는 임시적인 구조를 가집니다. 따라서 젠킨스 에이전트 작업 내용들은 삭제 전에 젠킨스 컨트롤러에 저장돼야 하며, 이를 위해 젠킨스 에이전트 서비스가 항상 동작하고 있습니다.

젠킨스 컨트롤러를 단독으로 설치할 경우에는 컨트롤러가 설치된 서버에서 젠킨스 자체 시스템 관리, CI/CD 설정, 빌드 등의 작업을 모두 젠킨스 컨트롤러 단일 노드에서 수행합니다. 하지만 컨트롤러-에이전트 구조로 설치할 경우 컨트롤러는 젠킨스 자체의 관리 및 CI/CD와 관련된 설정만을 담당하고 실제 빌드 작업은 에이전트로 설정된 노드에서 이루어 집니다. 


![젠킨스 컨트롤러-에이전트 구조](https://user-images.githubusercontent.com/22395934/235293824-4076b9ab-5e9d-4fa2-813e-b6bb8b6fe8cb.png)

## 4. jenkins 서비스 어카운트 권한 설정

젠킨스 에이전트 파드에서 쿠버네티스 API 서버로 통신하려면 서비스 어카운트에 권한을 줘야 합니다. 권한을 주기 전에 우선 jenkins 서비스 어카운트가 존재하는지 확인합니다.

```bash
kubectl get serviceaccounts
```

서비스 어카운트가 확인되면 쿠버네티스 클러스터에 대한 `admin` 권한을  부여합니다.

```bash
kubectl create clusterrolebinding jenkins-cluster-admin --clusterrole=cluster-admin --serviceaccount=default:jenkins
```

권한이 적용되었으면, 권한을 부여한 의미를 알아보았습니다.

jenkins 서비스 어카운트를 통해 젠킨스 에이전트 파드를 생성하거나 젠킨스 에이전트 파드 내부에서 쿠버네티스 오브젝트에 제약 없이 접근하려면 `cluster-admin 역할`을 부여해야 합니다. 

필요한 영역으로 나누어 권한을 부여하는 것이 일반적이나 효율적으로 학습하기 위해 cluster-admin 1개 권한만 부여했습니다.

서비스 어카운트에 `cluster-admin 역할`을 부여하고 이를 권한이 필요한 서비스 어카운트인 jenkins에 묶어줍니다. 이런 방식을 `역할 기반 접근 제어`라고 합니다.