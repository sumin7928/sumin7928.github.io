---

  layout: post
  title: CentOS7 에서 윈도우서버 원격 실행 하기
  date: 2018-07-24 16:00:00 +0900
  tags: centos windows
  description: winexe를 이용해서 원도우 서버의 스크립트 실행 

---

## 리눅스 환경

```
[root@localhost ~]# cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)
[root@localhost ~]# uname -a
Linux localhost.localdomain 3.10.0-957.12.1.el7.x86_64 #1 SMP Mon Apr 29 14:59:59 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

리눅스에서 윈도우 서버에 원격 접속할 수 있는 프로그램으로는 rdesktop, remmina등이 있다. 하지만 리눅스가 GUI가 아니라 따로 시도해 보지는 않았고
젠킨스를 이용해 스크립트 실행 목적으로 만들 예정이라 찾아보니 윈도우에서는 보통 psexec.exe 를 사용해서 윈도우 원격 제어 툴로 많이 사용되는 것으로 보인다. .

psexec.exe 사용을 위해서는 리눅스에서 winexe를 사용해야 하고 관련 소스는 공식 사이트에 제공되어있다.
https://sourceforge.net/p/winexe/winexe-waf/ci/master/tree/

## 설치

1. winexe 소스 가져옴
해당 사이트에서 제공되는 파일을 다운받던지 `git clone http://git.code.sf.net/p/winexe/winexe-waf winexe` 를 이용해서 전체 소스를 가져오면 된다.

2. 관련 패키지를 설치
```
yum install epel-release
yum update -y
yum install samba-client \
gcc \
perl \
mingw-binutils-generic \
mingw-filesystem-base \
mingw32-binutils \
mingw32-cpp \
mingw32-crt \
mingw32-filesystem \
mingw32-gcc \
mingw32-headers \
mingw64-binutils \
mingw64-cpp \
mingw64-crt \
mingw64-filesystem \
mingw64-gcc \
mingw64-headers \
libcom_err-devel \
popt-devel \
zlib-devel \
zlib-static \
glibc-devel \
glibc-static \
python-devel \
gnutls-devel \
libacl-devel \
openldap-devel \
samba-devel \
```

3. 컴파일 및 빌드 진행
받은 winexe 폴더의 source 폴더에서 진행
```
[root@localhost source]# ./waf configure build
```

아무 이상없이 빌드가 완료되면 문제가 없겠지만 하라는대로 했더니 문제가 발생했다. 

```
[root@localhost source]# ./waf configure build
Setting top to                           : /root/winexe/source
Setting out to                           : /root/winexe/source/build
Checking for 'gcc' (c compiler)          : /usr/bin/gcc
Checking for program pkg-config          : /usr/bin/pkg-config
Checking for 'dcerpc'                    : yes
Checking for 'talloc'                    : yes
SAMBA_INCS set to                        : /usr/include/samba-4.0
SAMBA_LIBS set to                        : []
Checking for samba_util.h                : no
Build of shared winexe                   : disabled
Cannot continue! Please either install Samba shared libraries and re-run waf, or download the Samba source code and re-run waf with the "--samba-dir" option.
(complete log in /root/winexe/source/build/config.log)
```

위의 문제가 발생해서 관련 라이브러리들을 막 설치해보고 공식 문서를 보니 `git clone git://git.samba.org/samba.git samba` 로 Samba 소스도 가져와
--samba-dir 옵션으로 해당 path를 지정했는데도 안되었다.

```
[root@localhost source]# ./waf --samba-dir=../../samba configure build
Setting top to                           : /root/winexe/source
Setting out to                           : /root/winexe/source/build
Checking for 'gcc' (c compiler)          : /usr/bin/gcc
Checking for program /root/samba/buildtools/bin/waf : /root/samba/buildtools/bin/waf
/usr/bin/env: python3: 그런 파일이나 디렉터리가 없습니다
Checking for library smb_static                     : not found
Build of static winexe                              : disabled
Cannot continue! Please either install Samba shared libraries and re-run waf, or download the Samba source code and re-run waf with the "--samba-dir" option.
(complete log in /root/winexe/source/build/config.log)
```

슬슬 맨붕이 오고 있을때쯤 좀더 찾아보니 Samba 버전에 대한 이슈였고 해당 이슈에 대해서는 누군가가 winexe를 fork해서 수정한 소스를 올린게 있어서
얼른 받고 다시 빌드를 해보았다.

```
[root@localhost source]# ./waf configure build
Setting top to                           : /root/winexe/source
Setting out to                           : /root/winexe/source/build
Checking for 'gcc' (c compiler)          : /usr/bin/gcc
Checking for program pkg-config          : /usr/bin/pkg-config
Checking for 'dcerpc'                    : yes
Checking for 'talloc'                    : yes
SAMBA_INCS set to                        : /usr/include/samba-4.0 ../smb4private/ ../
SAMBA_LIBS set to                        : /usr/lib64/samba
Checking for core/error.h                : yes
Checking for credentials.h               : yes
Checking for dcerpc.h                    : yes
Checking for gen_ndr/ndr_svcctl_c.h      : yes
Checking for popt.h                      : yes
Checking for tevent.h                    : yes
Checking for util/debug.h                : yes
Checking for library cli-ldap-samba4     : not found
Checking for library :libcli-ldap-samba4.so.0 : not found
Build of shared winexe                        : disabled
Cannot continue! Please either install Samba shared libraries and re-run waf, or download the Samba source code and re-run waf with the "--samba-dir" option.
(complete log in /root/winexe/source/build/config.log)
```

하지만 또 다른 이슈.. 이번엔 cli-.. 관련 라이브러리가 없다고하네.. yum 으로 찾아봐도 없고 계속 검색해도 없었지만 또 누군가가 우릴 위해서 삽질한 경험을 공유해주었다.

`[root@localhost source]# ln -s /usr/lib64/samba/libgenrand-samba4.so /usr/lib64/libgenrand-samba4.so`

라이브러리 링크가 제대로 안되어있는거고 심볼릭 링크를 이용해서 강제로 해주니!!

```
Waf: Leaving directory `/root/winexe/source/build'
'build' finished successfully (0.999s)
```

정상적으로 빌드가 되어 이제 winexe 파일을 실행할 수 있게 되었다.










