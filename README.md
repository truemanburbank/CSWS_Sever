# 1. 서버 등록하기

HOST 서버를 CSWS에 등록했을 때 성공 여부를 판단하기 위해 필요한 기능입니다. HOST에 직접 ssh 원격 명령어를 날려보냄으로써 연결 여부를 확인합니다.

# 2. 서버 자동화 파일

HOST는 CSWS에 연결을 요청한 후, wget으로 서버 자동화에 필요한 Automation 압축 파일을 다운로드 받을 수 있습니다. Automation.tar의 파일 내용은 다음과 같습니다.

![1](https://github.com/user-attachments/assets/8b7091ea-05ac-4495-afe1-aed0ef6c214e)

[Automation.tar와 그 안에 존재하는 파일들]

### id_rsa.pub

CSWS의 공개키입니다. HOST가 Automation.tar의 압축을 풀고 [ServerAutomation.sh](http://ServerAutomation.sh) 파일을 실행할 경우, id_rsa.pub이 HOST의 authorized_keys에 등록되어 CSWS와 HOST의 ssh 연결을 가능하게 합니다.

### logo.txt

CSWS의 로고 텍스트 파일입니다. 서버 등록이 확인될 경우 logo.txt가 HOST의 화면에 보입니다.

### nginxTemplet

Nginx의 config파일 수정을 위한 기본 템플릿입니다. 후에, 사용자가 도메인 등록을 할 경우 이 템플릿을 토대로 config 파일을 작성하게 됩니다.

### sh 폴더의  쉘 (H_AddInbound.sh ~ HostException.sh)

사용자가 CSWS 홈페이지를 통해 작업을 하면 CSWS에서 원격 쉘을 통해 사용자의 HOST로 ssh 원격 명령어를 보내게 되는데, 그때 실행되는 쉘입니다.

### ServerAutomation.sh

HOST는 해당 쉘을 실행하여 컴퓨터를 서버로 자동화할 수 있습니다. 쉘 내용은 다음과 같습니다.

1. 도커 컨테이너 옵션을 위한 xfs 옵션 설정
2. ssh  연결을 위한 openssh-server 설치
    1. config 파일 수정을 통한 ssh 연결 옵션 설정
    2. 기본 포트 22로 설정
    3. CSWS의 공개키 등록
    4. sshd 재시작
3. 서버 자원량을 알기 위한 sysstat 설치
4. 도커 설치 후 도커의 경로 바꾸기
    1. xfs로 포맷된 곳으로 도커 경로를 수정해야 도커 컨테이너 옵션을 적용할 수 있습니다.
5. iptables 설치 후 방화벽 설정
    1. 서버의 보안과 CSWS와의 ssh 연결을 위해 방화벽을 설정합니다.
6. 컨테이너의 도메인 등록을 위한 nginx 설치
7. 설정 적용을 위한 재부팅

# 3. CSWS에서 사용할 도커파일 제작

사용자는 CSWS에서 발급받은 개인키를 통해 HOST의 컨테이너에 접속할 수 있습니다. 그러나 기존 도커 이미지를 실행시킨 도커 컨테이너로는 개인키를 가지고 바로 접속하기가 불가능하므로 CSWS만을 위한 도커파일을 제작하였습니다.

![2](https://github.com/user-attachments/assets/c0bbbc16-11c2-4530-95c2-14dfb7ce4602)

![3](https://github.com/user-attachments/assets/00202151-73d9-4c87-b843-b3833297581e)


### 도커 파일의 내용(jamesclerkmaxwell/csws_ubuntu:0.71, jamesclerkmaxwell/csws_centos:0.71)

1. 사용자 기본 계정을 “csws”로 생성합니다.
2. ssh 연결을 위한 openssh-server를 설치합니다.
3. sshd_config 파일을 수정합니다.
    1. PubkeyAuthentication yes
    2. PasswordAuthentication no
    3. AutorizedKeysFile 경로 설정
    4. 기본 포트 22로 설정
4. autorized_keys 파일 생성
5. sshd 실행

# 4. CSWS 쉘

원격 쉘은 HOST에 ssh 원격 명령어를 보낼 때 사용하는 쉘입니다. 원격 명령어를 실행시켜 HOST 컴퓨터 내부에 있는 호스트 쉘을 실행시켜 HOST 상의 도커 컨테이너들을 제어합니다.

원격 쉘들은 모두 호스트 계정의 이름과 호스트 아이피를 필요로 하며, 그 외에 쉘마다 필요한 인수를 갖습니다.

1. 인스턴스 제어 쉘(생성, 시작, 중지, 재부팅, 종료)
| 파일 | 설명 | 
| --- | --- | 
| CreateContainer.sh | 도커 컨테이너를 생성하는 쉘을 원격으로 실행합니다.  |
| RemoveContainer.sh | 도커 컨테이너를 삭제하는 쉘을 원격으로 실행합니다. |
| RestartContainer.sh | 도커 컨테이너를 재시작하는 쉘을 원격으로 실행합니다. |
| StartContainer.sh | 도커 컨테이너를 시작하는 쉘을 원격으로 실행합니다. |
| StopContainer.sh | 도커 컨테이너를 정지하는 쉘을 원격으로 실행합니다. |
2. 인바운드 규칙 편집 쉘

| AddInbound.sh | 도커 컨테이너의 인바운드 규칙을 추가하는 쉘을 원격으로 실행합니다. |
| --- | --- |
| DeleteInbound.sh | 도커 컨테이너의 인바운드 규칙을 삭제하는 쉘을 원격으로 실행합니다. |
3. 도메인 적용 쉘

| AddNginx.sh | 도메인 적용을 위해 Nginx config 파일을 추가하는 쉘을 원격으로 실행합니다. |
| --- | --- |
| DeleteNginx.sh | 도메인 적용 해제를 위해 Nginx config 파일을 삭제하는 쉘을 원격으로 실행합니다. |
4. 출력 
- 컨테이너 목록

| PrintStatusforManager.sh | 해당 서버의 모든 컨테이너 목록을 출력합니다. 관리자 권한으로만 실행 가능합니다. |
| --- | --- |
| PrintStatusforUser.sh | 사용자 소유의 모든 컨테이너 목록을 출력합니다. |
- 서버 자원 사용량, 컨테이너 자원 사용량 출력

| CheckContainerResource.sh | 컨테이너의 자원 사용량을 확인합니다. 호스트에 컨테이너의 자원 사용량을 확인하는 원격 명령어를 보낸 후 해당 내용이 적힌 파일을 CSWS로 가져옵니다. |
| --- | --- |
| CheckServerResource.sh | 서버의 자원 사용량을 확인합니다. 호스트 전체의 자원 사용량을 확인하는 원격 명령어를 보낸 후 해당 내용이 적힌 파일을 CSWS로 가져옵니다. |
5. 연결 확인 쉘

| IsConnected.sh | 첫 연결 시, 호스트에 ssh 원격 명령어를 보내 호스트와 CSWS 사이 원격 명령어가 전달 가능한지 확인합니다. |
| --- | --- |
6. SSH 키페어 관련 쉘

| CreateKeypairs.sh | 인스턴스(도커 컨테이너)를 생성하기 전 키페어를 생성하는 쉘입니다.  개인키의 형식은 pem입니다. 개인키는 사용자에게 보내집니다. |
| --- | --- |
| SendPublickey.sh | 사용자가 인스턴스 생성을 마치면 공개키를 호스트 서버 컨테이너에 전송하고, 컨테이너에 키를 등록하기 위해 호스트 내부에 존재하는 원격 쉘을 실행시킵니다. |

# 5. 호스트 쉘

호스트 쉘은 교수자의 컴퓨터 안에 존재하는 쉘입니다. 사용자가 홈페이지 조작을 통해 CSWS에 명령을 보내면 CSWS가 호스트 쉘을 원격으로 실행시킵니다.

1. 인스턴스 제어 쉘(생성, 시작, 중지, 재부팅, 종료)

| H_CreateContainer.sh | 인수를 입력받아 컨테이너를 생성해 실행시킵니다. 조작할 수 있는 변수는 호스트 포트, 컨테이너 포트, 컨테이너를 실행시키는 유저 이름, 유저 코드, 컨테이너 용량, 실행시킬 csws의 이미지입니다. 사용자가 ssh로 접속하기 위해 컨테이너와 이어지는 호스트의 포트도 엽니다. |
| --- | --- |
| H_RemoveContainer.sh | 컨테이너 이름을 입력받아 해당 컨테이너를 삭제합니다. 이때, 컨테이너가 중지되어 있지 않아도 강제로 삭제합니다. |
| H_RestartContainer.sh | 컨테이너 이름을 입력받아 해당 컨테이너를 재시작합니다. |
| H_StartContainer.sh | 컨테이너 이름을 입력받아 해당 컨테이너를 시작합니다. |
| H_StopContainer.sh | 컨테이너 이름을 입력받아 해당 컨테이너를 중지합니다. |
2. 인바운드 규칙 편집 쉘

| H_AddInbound.sh | 사용자가 추가한 인바운드 규칙을 토대로 호스트의 포트를 엽니다. 그리고 도커 컨테이너를 커밋한 후 컨테이너의 포트를 추가하여 다시 실행시킵니다. |
| --- | --- |
| H_DeleteInbound.sh | 사용자가 삭제한 인바운드 규칙을 토대로 호스트의 포트를 닫습니다. 그리고 도커 컨테이너를 커밋한 후 컨테이너의 포트를 삭제하여 다시 실행시킵니다. |
3. 도메인 적용 쉘

| H_AddNginx.sh | 사용자가 적용한 도메인을 컨테이너와 연결시키기 위해 호스트의 Nginx config 파일을 생성합니다. |
| --- | --- |
| H_DeleteNginx.sh | 호스트 내부의 Nginx config 파일을 삭제하여 사용자가 등록한 도메인을 삭제합니다. |
4. 출력 

| H_PrintStatusforManager.sh | 현재 호스트 상에 존재하는 도커 컨테이너 목록을 모두 출력합니다. |
| --- | --- |
| H_PrintStatusforUser.sh | 현재 호스트 상에 존재하는 특정 유저의 컨테이너 목록을 모두 출력합니다. |
5. SSH 키페어 관련 쉘

| H_SendPublickey.sh | CSWS로부터 전달받은 사용자의 공개키를 호스트에 존재하는 사용자의 컨테이너로 보내 authorized_keys에 등록합니다. |
| --- | --- |

![4](https://github.com/user-attachments/assets/8eb5c342-2b1e-447b-9a31-cb35c9d710ea)

