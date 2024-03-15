# aws-v2

### 목표
- 스크립트 작성으로 배포 자동화하기, 재배포 자동화하기

### ip
- 공인 ip는 외부 네트워크에 노출되는 ip로 다른 공인 ip와 연결 가능
- 사설 ip는 홈 네트워크에서 쓰이는 ip로 내부에서만 사용 가능
- 공유기는 적은 수의 공인 ip를 돌려가며 사용하므로 ip가 고정되지 못하고 가변 상태
- 이를 dhcp 할당이라 하며 고정 ip를 사용해야 하는 서버 운영 불가능
- aws에서 프리티어 서버에 고정 ip를 제공
- 단, ec2서버와 연결하여야 무료이고 할당만 받은 경우 유료

### 세션 연결
- mobaxterm을 통해 세션을 생성
- 세션 설정 시, 만들어둔 ec2서버의 퍼블릭 ip 주소와 발급받은 키를 세팅

### 환경 변수
```sh
# 환경 변수
$HOME

# 환경 변수 설정
# 세션이 종료되면 같이 제거
export $LOVE="i love you"

# 환경 설정 불변 설정
# .bashrc는 환경 변수를 저장하는 리눅스 설정 파일
# source 명령어는 강제 적용, 안한다면 재부팅해야 적용
vi ./.bashrc
export LOVE="i love you"
source ./.bashrc

# 환경변수 파일 설정
vi var.sh
# bash 문법이라고 표시
#!/bin/bash
GITHUB_ID="hansung1908"
PROJECT_NAME="aws-v2"
PROJECT_VERSION="0.0.1"
PROJECT_PID=""
JAR_PATH=""

export GITHUB_ID
export PROJECT_NAME
export PROJECT_VERSION
export PROJECT_PID
export JAR_PATH

# 환경변수 파일 실행
vi deploy.sh
#!/bin/bash
source ./var.sh
echo $GITHUB_ID

# 실행 권한 부여
chmod u+x deploy.sh
```
- 여러 파일에서 사용하기 위해 파일로 설정

# 환경변수 적용 범위
- ./bashrc - 어디에서나 적용
- 터미널 만들고 source 적용 - 터미널이 꺼지기 직전까지
- 쉘 스크립트(파일)로 변수를 만들고 다른 파일에서 실행하기 위해서는
- deploy.sh 파일이 실행되는 동안에만 변수를 사용할 수 있으면 됨
- 파일에 source 명령어 작성
