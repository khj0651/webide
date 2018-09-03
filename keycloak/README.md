# 빠른 설치
```
docker run -it \
  -e CHE_MULTIUSER=true \
  -e CHE_HOST=192.168.0.8 \
  -e CHE_PORT=8089 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/workspace/webide/che-multiuser:/data \
  eclipse/che:6.9.0 start
```

# Configuration
## Keycloak
Eclipse Che의 계정 및 인증체계 모듈인 Keycloak과 UAA 간 연계를 위한 설정

>http://192.168.0.8:5050/auth/admin/master/console  
>admin/admin  
>Realm : che

### SAML Client 생성
```
Clients > Add Client  
 > Client ID: uaa-saml  
   Client Protocol: saml  
   Client SAML Endpoint: http://192.168.0.8:8089/dashboard/#/
```
![alt text](https://github.com/khj0651/webide/blob/master/keycloak/1.add%20saml%20client.png)

#### 저장 후 상세정보 수정
```
Clients > uaa-saml 
 > Signature Algorithm : RSA_SHA1  
   SAML Signature Key Name : NONE  
   IDP Initiated SSO URL Name : uaa
```
![alt text](https://github.com/khj0651/webide/blob/master/keycloak/2.detail.png)

### Identity Provider (SAML v2.0) 생성
```
Identity Providers > Add identity provider  
Import External IDP Config > UAA의 IdP Metadata Xml 파일 선택 > Import  
 > Alias "uaa-idp"  
   Enabled "ON"  
   Store Tokens "ON"  
   Stored Tokens Readable "ON"  
  
   NameID Policy Format "Unspecified"  
   HTTP-POST Binding Response "ON"  
   HTTP-POST Binding for AuthnRequest "ON"  
   Want AuthnRequests Signed "ON"  
   Signature Algorithm "RSA_SHA1"  
   SAML Signature Key Name "NONE"  
   Force Authentication "ON"
```
![alt text](https://github.com/khj0651/webide/blob/master/keycloak/3.idp.png)

### Service Provider Metadata 가져오기
Idp(uaa-idp)의 Export tab에서 Download  
또는  
http://192.168.0.8:5050/auth/realms/che/broker/uaa-idp/endpoint/descriptor

이 Metadata xml파일에서 SingleLogoutService>와 <AssertionConsumerService>의 HTTP-POST binding uri에 /clients/<client-id>를 추가한다
여기서 client-id란 saml client 생성 시 입력한 IDP Initiated SSO URL Name 값
```
예)
// http://192.168.0.8:5050/auth/realms/che/broker/uaa-idp/endpoint
http://192.168.0.8:5050/auth/realms/che/broker/uaa->idp/endpoint/clients/uaa
```


## UAA
### IdP Metadata 가져오기
>https://uaa.paas.lc/saml/idp/metadata  
>  (참고: sp metadata는 https://uaa.paas.lc/saml/metadata)

### Service Provider 등록
먼저 admin client의 authorities에 sps.read, sps.write 있는지 확인
```
uaac target https://uaa.paas.lc --skip-ssl-validation
uaac token client get admin -s loa91vra0fa0dbcp0wl5
uaac token decode
```

없으면 추가
```
uaac client update admin --authorities "<existing-authorities> sps.read sps.write"
uaac client update admin --authorities "clients.read password.write clients.secret clients.write uaa.admin scim.write scim.read sps.read sps.write"
```

이후 create-saml-sp.sh을 사용하여 SP 등록
```
create-saml-sp.sh -n <your-sp-name> -m sp-metadata.xml -s <your-sp-entity-id> -i
create-saml-sp.sh -n che -m keycloak-sp-metadata.xml -s http://192.168.0.8:5050/auth/realms/che -i
```

확인
```
uaac curl /saml/service-providers --insecure
```


## Test
### 사용자 추가
```
uaac user add hyodroid --given_name Hyojin --family_name Kim --emails hyodroid@gmail.com -p 1234qwer
uaac group add zones.uaa.admin
uaac member add zones.uaa.admin hyodroid
uaac member add uaa.none hyodroid
```

### Initiate IDP Login Flow
>https://uaa.paas.lc/saml/idp/initiate?sp=http://192.168.0.8:5050/auth/realms/che
