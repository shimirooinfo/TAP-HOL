
# Tanzu Application Platform (TAP) 인터넷 차단 환경 설치
본 가이드는 vSphere 인터넷 차단 환경에서 TAP를 설치하는 방법에 대해 설명합니다. 설치 환경에 대한 가이드는 다음 버전을 기준으로 합니다.
- vSphere
- TKG 1.5.4 (vSphere with Tanzu는 [설치 사전 요구사항](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-prerequisites.html)을 참고)
- TAP 1.2.1
- 앱 빌드를 위한 Maven Repository 별도 구성하지 않음 (인터넷 연결 가능하다고 가정하며, 이 부분에 대한 인터넷 차단 환경 구성에 대한 설명은 추구 가이드를 업데이트 할 예정입니다.)
- 소스 저장소: GitHub

<br/>

## 0. 사전 준비 사항
- 점프박스 및 TKG 설치
  - [설치 방법](../tap/jumpbox-prepare.md)

> **_NOTE:_** 아래 가이드는 주어진 점프박스를 사용했을 때 기준으로 작성되었습니다. 일반적인 vSphere air-gap 환경에서의 설치는 다음 문서를 참고해 주시기 바랍니다.    [링크](https://github.com/tanzukorea/tanzu-install/blob/main/tap/airgapped/installation-on-vsphere.md)

<br/>

## 1. 설치 환경 구성
### 1) 환경 변수
설치를 위한 환경 변수를 다음과 같이 설정합니다.   
아래 설정된 환경변수 값은 사전 제공된 점프박스를 사용했을 때 기준으로 하며, 별도로 구축한 경우에는 변경이 필요합니다.   
```
export INSTALL_REGISTRY_USERNAME=admin
export INSTALL_REGISTRY_PASSWORD=VMware1!
export INSTALL_REGISTRY_HOSTNAME=harbor.tanzukr.com
export TAP_VERSION=1.2.1
```
템플릿에 사용된 관련 변수 정보
|변수명|설명|
|------|---|
|INSTALL_REGISTRY_HOSTNAME|사설 레지스트리 접속 FQDN|
|INSTALL_REGISTRY_USERNAME|사설 레지스트리 접속 ID|
|INSTALL_REGISTRY_PASSWORD|사설 레지스트리 접속 비밀번호|

> **_NOTE:_** 아래 환경변수의 버전은 빌드 서비스의 버전을 의미합니다. TAP 패키지와 함께 설치된 빌드 서비스 버전을 참고하여 지정합니다.

### 2) 이미지 레파지토리 구성
TAP에서 사용하는 이미지 레파지토리 주소는 다음과 같습니다.
|레지스트리 주소|설명|
|------|---|
|$INSTALL_REGISTRY_HOSTNAME/tap/tap-packages|TAP 설치 파일|
|$INSTALL_REGISTRY_HOSTNAME/tbs/build-service|빌드 서비스 이미지 파일|
|$INSTALL_REGISTRY_HOSTNAME/tap/full-tbs-deps-package-repo|빌드 서비스 디펜던시 설치 파일|
|$INSTALL_REGISTRY_HOSTNAME/tap/supply-chain|Supply Chain에서 사용, 빌드된 앱 이미지들이 여기에 저장됨|

<br/>

## 2. 레지스트리 로그인
점프박스 VM에서 다음의 레지스트리에 로그인을 합니다. 

### 1) 하버 레파지토리 웹 페이지 접속
```
https://harbor.tanzukr.com
```

### 2) TAP 설치 이미지를 저장할 사설 레지스트리 로그인
```
docker login harbor.tanzukr.com
```
<br/>

## 3. TAP 설치

### 1) 네임스페이스 생성
TAP 설치를 위해 클러스터에 접속 후 다음 명령어를 사용하여 네임스페이스를 생성합니다.
```
kubectl create ns tap-install
```

### 2) TAP 설치를 위한 레지스트리 및 패키지 추가
#### a. 레지스트리 시크릿 추가
```
tanzu secret registry add tap-registry --username ${INSTALL_REGISTRY_USERNAME} --password ${INSTALL_REGISTRY_PASSWORD} --server ${INSTALL_REGISTRY_HOSTNAME} --export-to-all-namespaces --yes --namespace tap-install 
```

#### b. 패키지 레지스트리 추가
```
tanzu package repository add tanzu-tap-repository --url ${INSTALL_REGISTRY_HOSTNAME}/tanzu-application-platform/tap-packages:${TAP_VERSION} --namespace tap-install
```

#### c. 패키지 레지스트리 확인
설치된 패키지 레지스트리의 status를 확인합니다. Reconcile succeed를 확인 후 다음 단계로 넘어갑니다.
```
tanzu package repository get tanzu-tap-repository --namespace tap-install
```

#### d. 설치 가능한 패키지 목록 확인
```
tanzu package available list --namespace tap-install 
```

#### e. TAP 패키지와 함께 설치할 수 있는 패키지 및 버전 확인
```
tanzu package available get tap.tanzu.vmware.com/${TAP_VERSION} --values-schema --namespace tap-install 
```

### 3) 빌드 서비스를 위한 레지스트리 및 패키지 추가

#### a. 패키지 레지스트리 추가
```
tanzu package repository add tbs-full-deps-repository --url ${INSTALL-REGISTRY-HOSTNAME}/tap/tbs-full-deps:$VERSION --namespace tap-install
```

#### b. 패키지 레지스트리 확인
```
tanzu package repository get tbs-full-deps-repository --namespace tap-install
```

### 4) TAP 설치를 위한 YAML 파일 생성
TAP 을 구성할 프로파일은 Full을 기준으로 작성하였으며, 아래 템플릿을 참고하여 YAML 파일을 생성합니다.
* YAML 파일 템플릿 참조: [링크](./tap-values.yaml)
* ca_cert_data에는 인증서 정보를 기입합니다.
* 템플릿에 사용된 관련 변수 정보
  |변수명|설명|
  |------|---|
  |INSTALL_REGISTRY_HOSTNAME|사설 레지스트리 접속 FQDN|
  |INSTALL_REGISTRY_USERNAME|사설 레지스트리 접속 ID|
  |INSTALL_REGISTRY_PASSWORD|사설 레지스트리 접속 비밀번호|
  |INGRESS-DOMAIN|TAP 구성에 사용할 인그레스 도메인|
  |GIT-CATALOG-URL|TAP 카탈로그를 구성할 YAML 파일이 저장되어 있는 GIT의 주소|

### 5) TAP 설치
다음 명령어를 사용하여 TAP 패키지를 설치합니다. tap-values.yaml 파일은 위의 3)번 단계에서 생성한 파일입니다.
```
tanzu package install tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file tap-values-vsphere.yaml -n tap-install
```

만일 TAP 패키지의 수정이 필요하면 다음 명령어를 사용하여 패키지를 수정합니다.
```
tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file tap-values.yaml -n tap-install
```

### 6) TAP 설치 확인
다음 명령어를 사용하여 TAP 패키지 설치를 확인합니다.
```
tanzu package installed list -n tap-install
```

### 7) 빌드 서비스 디펜던시 설치
다음 명령어를 사용하여 빌드 서비스 디펜던시 패키지를 설치합니다.
```
tanzu package install full-tbs-deps -p full-tbs-deps.tanzu.vmware.com -v $VERSION -n tap-install
```

### 8) TAP 및 빌드 서비스 디펜던시 설치 확인
```
tanzu package installed list -n tap-install
```
설치가 정상적으로 완료되었으면 다음과 같이 패키지 목록이 나타나고, Status에 "Reconcile Succeeded"를 확인할 수 있습니다.
![](../../images/tanzu-package-list.png)

<br/>

## 4. TAP GUI 접속 확인
설치가 완료 되었으면, TAP GUI 콘솔에 접속하여 설치가 정상적으로 이루어 졌는지를 확인해 봅니다. 위의 설치 과정에서 인그레스 도메인을 변경하였다면, 그 도메인에 맞게 접속해 봅니다.
```
http://tap-gui.tap.tanzukr.com
```

### 참고
인터넷 차단 환경의 자세한 설치 방법은 [여기](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-install-intro.html#install-air-gap)를 참고하시기 바랍니다. 

