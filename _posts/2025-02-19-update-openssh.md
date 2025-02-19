---
title: CentOS에서 OpenSSH 버전 업데이트하기
date: 2025-02-19 9:52:52 +/-TTTT
categories: [Linux]
tags: [CentOS, OpenSSH, OpenSSL]
---

사내 CentOS 7에서 OpenSSH 7.4를 사용하고 있는데, 보안적인 측면에서 업데이트가 필요하다고 하여 소스 컴파일 방식을 사용해 OpenSSH 9.8로 변경한 내용을 작성합니다.

|         | 기존  | 변경  |
| ------- | ----- | ----- |
| OpenSSH | 7.4   | 9.8   |
| OpenSSL | 1.0.2 | 1.1.1 |

### 0. 버전 확인

먼저 현재 OpenSSH 버전을 확인합니다.

```bash
ssh -V
>> OpenSSH_7.4p1, OpenSSL 1.0.2k-fips 26 Jan 2017
```

현재는 OpenSSH 7.4, OpenSSL 1.0.2를 사용하고 있습니다. OpenSSH 9.8 버전 이상을 사용하기 위해서는 OpenSSL 1.1.1 이상이 필요합니다. 따라서 OpenSSL 버전부터 업데이트를 진행합니다.

<br/>

### 1. OpenSSL 업데이트

OpenSSL 업데이트를 위해 필요한 패키지를 설치합니다.

의존성 문제로 인해 —skip-broken 옵션을 통해 필요한 것들만 설치하도록 진행했습니다.

```bash
yum groupinstall "Development Tools" --skip-broken -y
yum install perl-core libtool --skip-broken -y
```

최신 OpenSSL을 다운로드 및 설치합니다.

```bash
cd /usr/local/src
wget https://www.openssl.org/source/openssl-1.1.1u.tar.gz
tar -xzf openssl-1.1.1u.tar.gz
cd openssl-1.1.1u
```

설치한 OpenSSL 컴파일 및 설치를 진행합니다

```bash
./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib
make
make test
make install
```

새로 설치된 OpenSSL을 기본 OpenSSL로 설정하도록 심볼릭 링크를 지정합니다.

```bash
mv /usr/bin/openssl /usr/bin/openssl.bak  # 기존 설정 백업
ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl
```

라이브러리 경로 설정 및 환경 변수 설정을 진행합니다.

```bash
echo "/usr/local/ssl/lib" > /etc/ld.so.conf.d/openssl.conf
ldconfig -v
echo "export PATH=/usr/local/ssl/bin:$PATH" >> /etc/profile
source /etc/profile
```

작업이 완료되었고 업데이트 된 버전을 확인합니다.

```bash
/usr/local/ssl/bin/openssl version
>> OpenSSL 1.1.1u  30 May 2023
```
<br/>

### 2. OpenSSH 업데이트

OpenSSH 업데이트 전, 관련 의존성이 있는지 확인합니다. 없을 경우에는 설치해줍니다.

```bash
yum list installed | grep zlib-devel
yum list installed | grep openssl-devel
yum list installed | grep gcc
yum list installed | grep make

# 없는게 있을 경우 설치
# yum install gcc make zlib-devel openssl-devel
```

기존 설정을 백업해두고, 기존 버전을 삭제합니다.

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
vi /etc/ssh/sshd_config.bak

yum remove openssh openssh-server
```

새로운 버전을 설치합니다.

```bash
cd /usr/local/src
wget https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-9.8p1.tar.gz
tar -xzf openssh-9.8p1.tar.gz
```

OpenSSH 9.8 이상을 위해서는 사용하는 key 파일들의 권한 변경이 필요합니다. 권한이 다를 시에 아래 사진처럼 make install시에 에러가 뜨기에 권한을 변경해줍니다. 

![permission-err.png](/assets/img/update-openssh/permission-err.png){: width="700" height="400" }

```bash
ls -al /etc/ssh
chmod 600 /etc/ssh/ssh_host_ecdsa_key
chmod 600 /etc/ssh/ssh_host_ed25519_key
chmod 600 /etc/ssh/ssh_host_rsa_key
```

OpenSSH 컴파일 시 필요한 라이브러리를 추가해주고, 관련 설정 파일을 작성합니다.

`/etc/pam.d/sshd` 파일은 SSH를 통한 사용자 인증 방식을 정의하며,  로그인 과정에서 어떤 인증 모듈을 사용할지, 계정/세션 관리는 어떻게 할지 등을 설정합니다.

```bash
rpm -qa | grep pam-devel  #해당 라이브러리 있는지 먼저 확인
yum install pam-devel

vi /etc/pam.d/sshd  # 관련 설정 파일 추가
```

```bash
#%PAM-1.0
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
session    required     pam_selinux.so close
session    required     pam_loginuid.so
session    required     pam_selinux.so open
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
```
{: file="/etc/pam.d/sshd"}

컴파일 및 설치를 진행합니다. (OpenSSL 버전이 1.1.1이 낮을 경우 여기에서 버전 관련 에러가 발생합니다!)

설정 시에 꼭 위에서 설치한 pam과 함께 설정하도록 옵션을 지정해줍니다.

```bash
cd openssh-9.8p1
./configure --prefix=/usr --sysconfdir=/etc/ssh --with-ssl-dir=/usr/local/ssl --with-pam
make
make install

# make 시 오류 발생 후 다시 make할 경우 make clean
# make install
```

빌드 이후에 sshd가 pam 라이브러리를 의존하는지 확인해줍니다.

```bash
ldd /usr/sbin/sshd | grep pam
```

![pam.png](/assets/img/update-openssh/pam.png)

설정 파일 복원 및 업데이트합니다. 사실 기존 설정을 덮지는 않아서 안 해도 상관은 없으나, 확실하게 하기 위해 업데이트 합니다.

```bash
cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
```
SFTP를 사용 중이라면 관련 설정도 확인해야합니다.
해당 서버에서는 SFTP를 사용 중인데 관련 설정에서 ForceCommand에 SFTP 관련 경로를 지정해 준 부분이 있어서, 변경 시에 SFTP가 정상적으로 접속 되지 않았습니다.
따라서 관련 설정을 제거했습니다.

```bash
vi /etc/ssh/sshd_config
# sftp 접속 불가 이슈 해결하기 위해 SFTP의 ForceCommand 관련 부분 주석 처리
```

설정이 완료되었고, ssh 재시작 합니다. 버전 확인 시에 업데이트가 된 것은 확인했으나.. 갑자기 restart가 안되는 문제가 발생합니다. 

```bash
ssh -V
>> OpenSSH_9.8p1, OpenSSL 1.1.1u  30 May 2023

systemctl restart sshd
>> 에러 발생!!
```

![restart-err.png](/assets/img/update-openssh/restart-err.png)

OpenSSH를 소스에서 컴파일 설치를 할 경우 바이너리만 설치되고, systemd 서비스 파일은 자동으로 생성되지 않기 때문입니다. 따라서 수동으로 서비스 파일 생성이 필요합니다.

<br/>

### 3. OpenSSH 서비스 파일 생성 및 등록

sshd.service 파일은 systemd가 SSH 데몬을 정의하는 파일입니다. 

- 어떻게 시작 종료
- 시스템 부팅 시 자동 시작 여부
- 실패 시 재시작 여부

sshd.service 파일을 생성합니다.

```bash
[Unit]
Description=OpenSSH server daemon
After=network.target

[Service]
ExecStart=/usr/sbin/sshd -D
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```
{: file="/etc/systemd/system/sshd.service"}

새로 생성한 서비스를 systemd에 인식 시켜주고, 서비스를 시작합니다.

```bash
# 새로 생성한 서비스를 systemd에 인식
sudo systemctl daemon-reload

# 서비스 시작 및 부팅 시 자동 시작 설정
sudo systemctl start sshd
sudo systemctl status sshd
sudo systemctl enable sshd

```
<br/>

### 4. 최종 확인

새로운 세션으로 SSH 접속 여부, SFTP 접속 여부를 확인합니다.

```bash
sudo tail -f /var/log/secure
```


<br>

---
참고 자료
- [[Linux] 보안 취약점 개선을 위해 OpenSSH 업데이트하기 (CentOS)](https://hmw0908.tistory.com/117)
