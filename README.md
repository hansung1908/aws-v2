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

### 환경변수 적용 범위
- ./bashrc - 어디에서나 적용
- 터미널 만들고 source 적용 - 터미널이 꺼지기 직전까지
- 쉘 스크립트(파일)로 변수를 만들고 다른 파일에서 실행하기 위해서는
- deploy.sh 파일이 실행되는 동안에만 변수를 사용할 수 있으면 됨
- 파일에 source 명령어 작성

### 재배포를 고려하여 cron 종료
```sh
vi deploy.sh

#!/bin/bash

# 1. env variable
source ./var.sh
echo "1. env variable setting complete"

# 2. cron delete
touch crontab_delete
crontab crontab_delete
rm crontab_delete
echo "2. cron delete complete"
```
- ec2 서버에 프로젝트가 올라가 실행되면 포트가 할당되고 pid가 부여됨
- cron 파일은 자동으로 해당 pid에 접근하여 종료시 서버 재시작
- 재배포 시, 프로젝트 삭제 -> github 다운로드 -> jar 빌드 -> 실행 (약 30초 소요)
- 해당 과정에서 서버가 종료되지만 다시 실행할 필요가 없음
- 하지만 cron 파일로 인해 재시작하므로 일단 삭제하고 재배포가 종료되는 시점에 cron을 재등록

### 서버 pid 찾아서 종료시키기
- 빌드시 aws-v2-0.0.1.jar 형태로 실행 파일이 만들어짐
- gradle은 프로젝트에 포함됨 라이브러리를 관리하며 빌드시 라이브러리를 추가시켜 줌
- aws-v2-0.0.1-plain.jar은 gradle이 포함이 안된 실행파일로 같이 만들어짐
- 하지만 위 파일은 필요가 없으므로 프로젝트에 build.gradle 설정 파일에 아래의 명령어를 추가
```sh
jar {
  enable = false
}
```
- 서버 pid를 찾아 종료시키기 위해 var.sh의 남은 부분 채우기
```sh
# 프로젝트의 pid 찾기
# pgrep은 pid를 찾는 명령어, -f는 프로세스 이름이 패턴과 일치하는 대조하는 옵션
# pgrep -f aws-v2-0.0.1.jar
PROJECT_PID="$(pgrep -f ${PROJECT_NAME}-${PROJECT_VERSION}.jar)"

# 실행 파일의 위치
# /home/ubuntu/aws-v2/build/libs/aws-v2-0.0.1.jar
JAR_PATH="${HOME}/${PROJECT_NAME}/build/libs/${PROJECT_NAME}-${PROJECT_VERSION}.jar"
```
- 실제 스크립트 실행을 위해 deploy.sh 해당 내용 추가
```sh
# 3. server checking
# -n은 문자열 길이가 0이 아니면 참
# 해당 명령어가 참이라면 프로세스가 실행 중
if [ -n "${PROJECT_PID}" ]; then
  # re deploy
  kill -9 $PROJECT_PID
  echo "3. project kill complete"
else
  # first deploy

  # 3-1 apt update
  # bash 이용하므로 apt를 apt-get으로 수정
  # -y는 설치여부를 무조건 yes로 선택한다는 옵션
  # 1>/dev/null은 설치시 나오는 진행상황을 띄우지 않고 버린다는 명령어
  sudo apt-get -y update 1>/dev/null
  echo "3-1. apt-get update complete"

  # 3-2 jdk install
  sudo apt-get -y install openjdk-11-jdk 1>/dev/null
  echo "3-2. jdk install complete"

  # 3-3 timezone
  sudo timedatectl set-timezone Asia/Seoul
  echo "3-3. timezone setting complete"
fi
```

### 프로젝트 다운로드 및 빌드
- deploy.sh에 해당 내용 추가
```sh
# 4. project folder delete
# 프로젝트 다운 전 기존에 있던 폴더와의 충돌을 피하기 위해 기존 폴더 삭제
# rm -rf /home/ubuntu/aws-v2
rm -rf ${HOME}/${PROJECT_NAME}
echo "4. project folder delete complete"

# 5. git clone
# 프로젝트 세팅
git clone https://github.com/${GITHUB_ID}/${PROJECT_NAME}.git
# 확실하진 않지만 apt-get update, install처럼 동기식(일의 순서가 존재)처럼 동작하지 않고
# 비동기식으로 동작하므로 clone의 실행을 보장해주기 위해 3초 정도의 대기시간 부여
sleep 3s
echo "5. git clone complete"

# 6. gradle +x
# gradle을 통해 빌드하기 위해 실행권한 부여
chmod u+x ${HOME}/${PROJECT_NAME}/gradlew
echo "6. gradle u+x complete"

# 7. build
cd ${HOME}/${PROJECT_NAME}
./gradlew build
echo "7. gradlew build complete"
```
- 추가 후 직접 실행을 통한 테스트
```shell
# jar 파일 위치로 이동
cd /aws-v2/build/libs

# jar 실행
# 그냥 실행할 경우 application-dev.yml로 설정되어 있어 8081 포트를 통해 접속 가능
java -jar aws-v2-0.0.1.jar

# application-prod.jar로 변경하여 8080 포트로 접속
java -jar -Dspring.profiles.active=prod aws-v2-0.0.1.jar
```
- 접속 주소는 aws에서 설정한 static ip에 :8080/aws/v2를 붙혀서 접속