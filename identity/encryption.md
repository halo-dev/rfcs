## 动机

在安全性方面，为了防止请求伪造和数据泄露经常需要很多加密方式来阻止这一切，Halo也如此，我们需要一种更为安全的方式来传输数据和身份验证。

## 目标和非目标

- 提出针对后台用户身份确认的密钥机密机制
- 提出针对自定义API token的安全的验证机制
- 提出用户密码的机密方式，防止密码泄露
- 提出一种可以减少伪造密钥对系统资源占用的方案

## 设计

### Api token

如果用户不配置`salt`则在安装时在工作空间创建名为`apiToken.salt`的文件，使用如下代码生成`salt`。

```java
BytesKeyGenerator bytesKeyGenerator = KeyGenerators.secureRandom(16);
String salt = new String(Hex.encode(bytesKeyGenerator.generateKey()));
```

通过CRC32对生成的`API_TOKEN+salt`进行签名后拼接到`API_TOKEN`后在对其进行`Base62`编码

```java
public class EncryptionTest {
  // 安装时生成的salt
  private final String salt = "1c4a4c069de3e9d16bdc300f8af21a36";

  @Test
  public void test() {
    // 生成32位的随机api token
    String apiToken = new String(Hex.encode(KeyGenerators.secureRandom(16).generateKey()));
    // apiToken + salt并生成crc32
    String checksum = crc32((apiToken + salt).getBytes());
    // 将checksum拼接
    String finalApiToken = Base62.encode(apiToken + checksum);
    
    //最终结果示例: 4IuENoNS1YaVJlVs8KetWnWQXJLKyobzncU6zIBf7Y4MjjDtLOC2pN
  }

  public String crc32(byte[] array) {
    CRC32 crc32 = new CRC32();
    crc32.update(array);
    return Long.toHexString(crc32.getValue());
  }
}
```

### Jwt token

配置`JwtDecoder`和`JwtEncoder`使用`RSAKey`

```java
@Configuration
public class SecurityConfiguration {

	@Value("${jwt.public.key}")
	RSAPublicKey key;

	@Value("${jwt.private.key}")
	RSAPrivateKey priv;
  
  @Bean
	JwtDecoder jwtDecoder() {
		return NimbusJwtDecoder.withPublicKey(this.key).build();
	}

	@Bean
	JwtEncoder jwtEncoder() {
		JWK jwk = new RSAKey.Builder(this.key).privateKey(this.priv).build();
		JWKSource<SecurityContext> jwks = new ImmutableJWKSet<>(new JWKSet(jwk));
		return new NimbusJwtEncoder(jwks);
	}
}
```
配置密钥位置
```yaml
jwt:
  private.key: classpath:app.key
  public.key: classpath:app.pub
```

使用实例：

```java
@RestController
public class TokenController {

	@Autowired
	JwtEncoder encoder;

	@PostMapping("/token")
	public String token(Authentication authentication) {
		Instant now = Instant.now();
		long expiry = 36000L;

		String scope = authentication.getAuthorities().stream()
				.map(GrantedAuthority::getAuthority)
				.collect(Collectors.joining(" "));
		JwtClaimsSet claims = JwtClaimsSet.builder()
				.issuer("self")
				.issuedAt(now)
				.expiresAt(now.plusSeconds(expiry))
				.subject(authentication.getName())
				.claim("scope", scope)
				.build();

		return this.encoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
	}
}
```

### Password

```java
@Test
public void test() {
  // spring security默认的 strength 为 10，也可以使用 -1 表示使用默认值
  int strength = 10;
  String plainPassword = "12345678";
  // 使用 SecureRandom 作为盐生成器，因为它提供了加密强的随机数
  BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder(strength, new SecureRandom());
  // 加密密码
  String encodedPassword = bCryptPasswordEncoder.encode(plainPassword);
  
  System.out.println(encodedPassword);

  // 密码匹配
  boolean matches = bCryptPasswordEncoder.matches(plainPassword, encodedPassword);
	assertThat(matches).isTrue();
}
```

## 考虑的替代方案

[Generate a Secure Random](https://www.baeldung.com/java-generate-secure-password)

[AES Encryption and Decryption](https://www.baeldung.com/java-aes-encryption-decryption)

[Java Security Standard Algorithm Names](https://docs.oracle.com/en/java/javase/13/docs/specs/security/standard-names.html)
