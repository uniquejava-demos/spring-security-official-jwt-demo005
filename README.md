In my [previous demo](https://github.com/uniquejava-demos/spring-security-custom-auth-provider-demo004), we used nimbus
jose jwt shared secret key to generate jwt, in this demo, we will use public/priv key to decode/encode jwt.

This demo is based
on [https://github.com/spring-projects/spring-security-samples](https://github.com/spring-projects/spring-security-samples/tree/main/servlet/spring-boot/java/jwt/login)

## Environment

- Java 17
- Spring Boot 2.7.4

## Notes

1. spring resource server可以单独存在， 因为resource server负责提供rest api， 这个api可以是token endpoint， 由此实现自给自足。
2. spring resource server内置了JwtTokenFilter， 不需要像上一个demo那样自己手写。
3. resource server需要jwt decoder来validate token， 有3中配置decoder的方式
    - public key
    - shared secret key
    - jwk set uri
      ![](./doc/images/rs-jwt-decoder.png)
4. 如果要自己实现token endpoint，还需要提供jwt encoder
5. httpBasic可以帮助我们快速实现 `/token` endpoint， 因为可以从controller里直接拿到 `Authentication`
6. 这样一来， 我们的Rest API既可以使用basic 认证， 也可以使用token 认证
7. demo中的exceptionHandling处理不是很合理(不够restful,需要自己定制)

这个demo让我对spring resource server和http basic认证的使用方式和使用场景有了新的认识。

## Test

```bash
$ curl -u cyper:password -w "\n" -X POST http://localhost:8080/token

$ curl -H 'Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJzZWxmIiwic3ViIjoiY3lwZXIiLCJleHAiOjE2NjYyODA5NDksImlhdCI6MTY2NjI0NDk0OSwic2NvcGUiOiJhcHAifQ.PQ_cPWXEiqYvHtiDxuFAE2y2OAB51quwepyoPWAy-w4HCTW8tzz6ORXaHsU2BRqm0aKcIxcnQJL49y0pCr2uTS6qCRsB7VM2us13CMghAT50ubIQcvpezDC5wHTQFwDLa5vB4zlLlzrmAh1poyb52pByOBsFXUHqZDAx-9ox3KnnVPAFdKVz7eJE33l2xkLdwyM8z3YnRJ9x5GQ4br2tg3jIGSbvDkKG6k88R6_lwN0W5JP0LODAUyST9P_xOY9793kE2ENKrsg1_XLbDg1FijurJU55TrHWPPWyyVGrDSFAQOoI_ZBVZl3-mLQLcB7sqzavVro2L9CDuwO6t3mumg' \
http://localhost:8080 && echo


$ curl -u cyper:password  http://localhost:8080 && echo

```

## Generate pub/priv keys with openssl

```bash
# create rsa key pair
openssl genrsa -out keypair.pem 2048

# extract public key
openssl rsa -in keypair.pem -pubout -out public.pem

# create private key in PKCS#8 format
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in keypair.pem -out private.pem
```

Refer to Dan's blog: https://www.danvega.dev/blog/2022/09/06/spring-security-jwt/, 里面有写如何对jwt做junit测试

use it

```java
@Bean
JwtDecoder jwtDecoder(){
    return NimbusJwtDecoder.withPublicKey(this.key).build();
}

@Bean
JwtEncoder jwtEncoder(){
   JWK jwk=new RSAKey.Builder(this.key).privateKey(this.priv).build();
   JWKSource<SecurityContext> jwks=new ImmutableJWKSet<>(new JWKSet(jwk));
   return new NimbusJwtEncoder(jwks);
}
```

## Random keys with java code
see https://docs.spring.io/spring-authorization-server/docs/current/reference/html/getting-started.html

```java
@Bean
public KeyPair keyPair() {
  KeyPair keyPair;
  try {
      KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
      keyPairGenerator.initialize(2048);
      keyPair = keyPairGenerator.generateKeyPair();
  } catch (Exception ex) {
      throw new IllegalStateException(ex);
  }
  return keyPair;
}

 @Bean
 JwtDecoder jwtDecoder() {
     return NimbusJwtDecoder.withPublicKey((RSAPublicKey) keyPair.getPublic()).build();
 }

 @Bean
 JwtEncoder jwtEncoder() {
     JWK jwk = new RSAKey.Builder((RSAPublicKey) keyPair.getPublic()).privateKey(keyPair.getPrivate()).build();
     JWKSource<SecurityContext> jwks = new ImmutableJWKSet<>(new JWKSet(jwk));
     return new NimbusJwtEncoder(jwks);
 }
```

## Todo

csrf.ignoringRequestMatchers报错，需要改成csrf.ignoringAntMatchers， 这应该是spring 6的breaking change之一。