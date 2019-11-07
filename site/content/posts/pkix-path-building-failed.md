---
author: "Kim, Geunho"
date: 2019-11-07
title: "Java - PKIX path building failed"
---

# 오류 발생 원인

SSL 연결이 필요한 웹 리소스를 웹 브라우저에서는 신뢰하는 인증서로 확인됨에도 자바 클라이언트에서 아래와 같은 오류가 발생할 때가 있다.

```
PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested
```

오류 메시지 내용대로, 요청에 대해 올바른 인증 경로를 찾지 못했다는 것인데 왜 브라우저에서는 확인된 인증서가 애플리케이션에서는 오류가 발생하는걸까?

그 답은 SSL 연결의 HANDSHAKE 과정을 살펴보면 알 수 있다.

1. 클라이언트가 CA[^1]로 부터 발급받은 인증서로 서비스허는 서버로 연결을 맺는다.
2. 서버가 전송한 인증서가 CA로 부터 발급받은 것인지 확인한다. 클라이언트는 CA와 공개키 목록을 가지고 있다.
3. 신뢰할 수 있는 CA로 부터 발급된 인증서로 확인이 되면 CA의 공개키로 인증서를 복호화한다.
4. 세션 키 값을 생성해서 인증서에 저장된 서버의 공개키로 암호화해서 서버로 전송한다.
5. 세션 키로 암호화된 요청을 전송한다.

오류가 발생하는 지점은 2번 단계로 클라이언트가 신뢰하는 CA 목록에 인증서가 발급된 CA가 없는 경우 발생하는 것이다.

다시 말해서 웹 브라우저에서 요청시 신뢰할 수 있는 인증서로 표기되는 서비스라면, 오류가 발생하는 자바 클라이언트가 가지고 있는 CA 목록이 오래되었다는 뜻이다.

따라서 위의 오류는 주로 오래된 버전의 자바 클라이언트에서 발생한다. HANDSHAKE 단계에서 발생한 오류이기 때문에 서버로는 어떠한 요청 유입도 되지 않아서 오류 로그도 남지 않는다.
오류는 연결을 시도한 클라이언트에만 남기 때문에 이러한 오류를 서버 측에서 먼저 인지하기는 어렵다.

# 해결 방법

자바 애플리케이션은 keystore 파일에 CA 목록을 유지한다.

```
$JAVA_HOME/lib/security/cacerts
# 혹은
$JAVA_HOME/jre/lib/security/cacerts
```

여기서 `$JAVA_HOME/bin/keytool` 도구로 CA 목록을 출력/추가/삭제할 수 있다.

```
keytool -list -v -keystore $JAVA_HOME/lib/security/cacerts | grep 'Owner:' | grep DIgiCert
Enter keystore password: # 패스워드는 default 값으로 'changeit'

Owner: CN=DigiCert Assured ID Root CA, OU=www.digicert.com, O=DigiCert Inc, C=US
Owner: CN=DigiCert High Assurance EV Root CA, OU=www.digicert.com, O=DigiCert Inc, C=US
Owner: CN=DigiCert Global Root CA, OU=www.digicert.com, O=DigiCert Inc, C=US
```

이제 오류가 발생하는 도메인의 인증서 정보를 내려받아보자.
`openssl` 도구로 인증서 체인 정보를 출력할 수 있다.

```
openssl s_client -connect blog.geunho.dev:443 -showcerts
```

출력 중 PEM 형식에 해당하는 `BEGIN CERTIFICATE` ~ `END CERTIFICATE` 내용을 복사해서 `my.crt` 파일에 내용을 기록한다.
여러 목록이 나타난다면 각각 다른 `.crt` 파일에 기록해서 저장한다.

이제 다시 `keytool` 도구로 `.crt` 파일을 통해 CA를 등록할 수 있다.

```
keytool -import -alias mycert -keystore $JAVA_HOME/lib/security/cacerts -file my.crt
Enter keystore password: # 패스워드는 default 값으로 'changeit'
```

애플리케이션을 재시작하면 이제 요청/응답을 잘 받는 것을 확인할 수 있다.

[^1]: [CA (Certificate Authority)](https://en.wikipedia.org/wiki/Certificate_authority), 인증서를 발급하는 신뢰할 수 있는 인증 기관
