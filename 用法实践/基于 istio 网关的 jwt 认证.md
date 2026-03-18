# Istio 网关 jwt 认证实现

在网关层面实现 jwt 的请求认证。istio 提供 RequestAuthentication 能力，原生支持 jwt 认证。

## 方案

### 架构

![架构][jwt-istio-gw]


### Jwks 服务
用于 istio ingress 网关获取配置，进而对 token 进行验证。需要运维来提供（需要注意私钥的保密）。
服务返回 jwks 格式：

```json
{
  "keys": [
    {
      "kty": "EC",
      "crv": "P-256",
      "kid": "test-01",
      "x": "odQEPSWEyqlVYU9svYeL8c3r9EQaw1hbkLwAOxq0XNg",
      "y": "q6_Pt4bEIMv2bN8-ycCEZHLbh6xRDjJIyZsPSbgLHeQ",
      "use": "sig",
      "alg": "ES256"
    }
  ]
}
```

字段解释如下：

| 字段 | 解释 |
|------|------|
| use  | 表示密钥的用途。常见的值为 "sig"，表示该密钥用于签名。 |
| kty  | 表示密钥的类型。常见的值为 "EC"，表示椭圆曲线公钥。 |
| kid  | 密钥 ID，用于标识密钥。它在 JWT 的验证过程中帮助选择正确的密钥进行验证。 |
| crv  | 表示椭圆曲线的名称。常见的值为 "P-256"，表示使用 P-256 椭圆曲线。 |
| alg  | 表示算法类型。常见的值为 "ES256"，表示使用 ECDSA（椭圆曲线数字签名算法）的 SHA-256 变体。 |
| x/y  | 表示椭圆曲线密钥的坐标值。这些值用于计算签名和验证 JWT 的有效性。 |

### Jwt Token 生成服务
需要业务提供初步认证的服务。


### 配置 RequestAuthentication

1. Issuer 需要与业务配置是否配置
2. jwksUri 的服务为：http://jwks.secret.svc.cluster.local/jwt/jwks.json
3. 通过 Authorization http 头获取 Token
4. 验证通过解码后的内容通过 outputPayloadToHeader 头来传递到后端
5. 通过 outputClaimToHeaders 配置将 payload 中的 key 对应的值复制给响应的自定义头

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: "auth-jwt-gw"
  namespace: istio-ingress
spec:
  selector:
    matchLabels:
      istio: ingressgateway-default
  jwtRules:
  - issuer: "testing@yw-opt"
    jwksUri: "http://jwks.secret.svc.cluster.local/jwt/jwks.json"
    fromHeaders:
    - name: Authorization
      prefix: 'Bearer '
    outputPayloadToHeader: x-authenticated-token-payload
    outputClaimToHeaders:
    - header: x-jwt-group
      claim: group
    - header: x-jwt-user
      claim: user
```

### 配置 AuthorizationPolicy

配置 AuthorizationPolicy 明确哪些请求需要认证，哪些不需要认证
1. 允许 /auth 和 /public 请求不认证
2. 其他所有请求都需要 Token 认证

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: auth-jwt-policy
  namespace: istio-ingress
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals:
        - '*'
    to:
    - operation:
        paths:
        - /*
  - to:
    - operation:
        paths:
        - /auth
        - /public
  selector:
    matchLabels:
      istio: ingressgateway-default
```

## 脚本工具

python 生成 jwks 和 token 脚本

```python
# 使用 cryptography 生成一个 jwks 文件
# 用于签名，秘钥类型为 EC，算法类型为 ES256，椭圆曲线为 P-256
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import serialization
import base64
import json
import jwt
from datetime import datetime, timedelta

def generate_jwks():
    # 生成 EC 私钥
    private_key = ec.generate_private_key(ec.SECP256R1(), default_backend())
    
    # 获取公钥
    public_key = private_key.public_key()
    
    # 导出公钥为 PEM 格式
    public_pem = public_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )
    
    # 导出私钥为 PEM 格式
    private_pem = private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.TraditionalOpenSSL,
        encryption_algorithm=serialization.NoEncryption()
    )
    
    # 创建 JWKS 对象
    jwks = {
        "keys": [
            {
                "kty": "EC",
                "crv": "P-256",
                "x": jwt.utils.base64url_encode(public_key.public_bytes(
                    encoding=serialization.Encoding.X962,
                    format=serialization.PublicFormat.UncompressedPoint
                )[1:33]).decode('utf-8'),
                "y": jwt.utils.base64url_encode(public_key.public_bytes(
                    encoding=serialization.Encoding.X962,
                    format=serialization.PublicFormat.UncompressedPoint
                )[33:65]).decode('utf-8'),
                "use": "sig",
                "alg": "ES256"
            }
        ]
    }
    
    return jwks, private_pem.decode('utf-8')

# 生成 JWT 的函数
def generate_jwt(private_key_pem):
    # 定义 JWT 的头部和载荷
    headers = {
        "alg": "ES256",
        "typ": "JWT"
    }
    payload = {
        "sub": "user123",  # 用户 ID
        "name": "John Doe",  # 用户名
        "iat": datetime.utcnow(),  # 签发时间
        "exp": datetime.utcnow() + timedelta(hours=1)  # 过期时间
    }

    # 使用私钥生成 JWT
    token = jwt.encode(payload, private_key_pem, algorithm="ES256", headers=headers)
    return token

# 验证 JWT 的函数
def verify_jwt(token, public_key_pem):
    try:
        # 使用公钥验证 JWT
        decoded = jwt.decode(token, public_key_pem, algorithms=["ES256"])
        return decoded
    except jwt.ExpiredSignatureError:
        print("Token has expired")
    except jwt.InvalidTokenError:
        print("Invalid token")
    return None

# 使用 jwks 验证 JWT
def verify_jwt_with_jwks(token):
    with open("jwks.json", "r") as f:
        jwks = json.load(f)
    
    public_key = jwt.algorithms.ECAlgorithm.from_jwk(json.dumps(jwks["keys"][0]))
    
    decoded = verify_jwt(token, public_key)
    
    if decoded:
        print("JWT is valid. Decoded payload:", decoded)
    else:
        print("JWT verification failed.")

if __name__ == "__main__":
    jwks, private_key_pem = generate_jwks()
    
    # 打印 JWKS 和私钥
    print("JWKS:")
    print(json.dumps(jwks, indent=2))
    
    print("\nPrivate Key PEM:")
    print(private_key_pem)
    
    # 将 JWKS 保存到文件
    with open("jwks.json", "w") as f:
        json.dump(jwks, f, indent=2)

    token = generate_jwt(private_key_pem)
    print("\nGenerated JWT:", token)

    verify_jwt_with_jwks(token)
```

## 参考

1. [rfc-JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
2. [Istio 验证环境](https://istio.io/latest/docs/tasks/security/authentication/authn-policy/)
3. [Istio Authorization Policy 配置](https://istio.io/latest/docs/reference/config/security/authorization-policy/)

[jwt-istio-gw]: /images/jwt-istio-gw.jpeg