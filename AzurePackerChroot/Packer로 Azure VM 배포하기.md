# Packer 의 chroot 로 Azure VM 만들기



[TOC]



## 0. Overview

###  chroot vs arm

`chroot`빌더는 Azure Managed Disk(이하 MD) 이미지로 빌드가 가능합니다. `azure-arm`빌드와의 차이점은 매 빌드 마다 새 Azure VM을 시작하지 않고 이미 실행중인 Azure VM을 사용하여 관리 디스크 이미지를 빌드할 수 있다는 것 있니다.



### Prerequisite

- Azure 계정
- Python
- PIP



## 1. Install&SetUp

### Install Packer

```bash
# Python 가상 환경 설치
$ sudo apt install python3.11-venv

# Python 가상 환경 생성
$ python3 -m venv packerenv

# 가상 환경 폴더로 이동
$ cd ./packerenv

# Python 가상 환경 활성화
$ source ./bin/activate

# HashCorp GDG 키 추가
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -

# 공식 HashiCorp Linux 리포지토리 추가
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

# 패키지 업데이트 & packer 설치
sudo apt-get update && sudo apt-get install packer

packer plugins install github.com/hashicorp/azure
```

## 2. Configure Azure Credentials

Azure 에서 Contributor 권한이 있는 Service Principal을 생성 합니다.

`az login`

`az ad sp create-for-rbac --role Contributor --scopes /subscriptions/<subscription_id> --query "{ client_id: appId, client_secret: password, tenant_id: tenant }" `

```bash
{
  "client_id": "38d8086f-96d7-4619-997f-910783a6b938",
  "client_secret": "j8F.nNVHC3pV2D.zhu4BwjeKpOVHgn6OY4",
  "tenant_id": "16b3c013-d300-468d-ac64-7eda0820b6d3",
  "Subscription_id" : 4b8ce703-eca5-4362-97b8-03863e238e53
}
```

## 3. Create a Packer Template

hcl을 이용했을 때는 계속 오류가 발생하여 포기하고 json으로 작성하였습니다.

azure-ubuntu.json

```bash
{
  "variables": {
    "client_id": "{{env `ARM_CLIENT_ID`}}",
    "client_secret": "{{env `ARM_CLIENT_SECRET`}}",
    "subscription_id": "{{env `ARM_SUBSCRIPTION_ID`}}",
    "resource_group": "{{env `ARM_IMAGE_RESOURCEGROUP_ID`}}"
  },
  "builders": [
    {
      "type": "azure-chroot",
      "client_id": "{{user `client_id`}}",
      "client_secret": "{{user `client_secret`}}",
      "subscription_id": "{{user `subscription_id`}}",
      "image_resource_id": "/subscriptions/{{user `subscription_id`}}/resourceGroups/{{user `resource_group`}}/providers/Microsoft.Compute/images/MyUbuntuOSImage-{{timestamp}}",
      "source": "canonical:0001-com-ubuntu-server-jammy:22_04-lts:latest"
    }],
  "provisioners": [
    {
      "inline": ["apt-get update", "apt-get upgrade -y"],
      "inline_shebang": "/bin/sh -x",
      "type": "shell"
    }
  ]
}
---
export ARM_CLIENT_ID="38d8086f-96d7-4619-997f-910783a6b938"
export ARM_CLIENT_SECRET="j8F.nNVHC3pV2D.zhu4BwjeKpOVHgn6OY4"
export ARM_SUBSCRIPTION_ID="4b8ce703-eca5-4362-97b8-03863e238e53"
export ARM_IMAGE_RESOURCEGROUP_ID="test-rg"

sudo -E packer build azure-ubuntu.json
sudo /usr/bin/packer build azure-ubuntu.json
```



## 4. Create VM from Azure Image

``` bash
az vm create \
   --resource-group test-rg \
   --name pck-test-ubuntu \
   --image MyUbuntuOSImage-1700469038\
   --admin-username ubuntu \
   --generate-ssh-keys
```



