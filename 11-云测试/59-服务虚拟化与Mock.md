# 59-服务虚拟化与Mock
> 课时：50 min | 难度：★★★☆☆

## 学习目标
- 理解服务虚拟化的概念、价值与适用场景
- 掌握 WireMock 进行 HTTP 服务 Stub 的完整方法
- 掌握 Mountebank 进行多协议服务虚拟化
- 理解消费者驱动契约测试（CDC）与 Pact 实现
- 能够在微服务测试中设计 Mock 策略

## 核心概念

### 服务虚拟化 vs Mock vs Stub

| 概念 | 粒度 | 用途 | 工具示例 |
|------|------|------|----------|
| Mock | 方法/函数级别 | 单元测试中替换依赖 | Mockito, unittest.mock |
| Stub | 接口级别 | 返回固定响应 | WireMock |
| 服务虚拟化 | 服务级别 | 模拟完整服务行为 | Mountebank, WireMock |

### 服务虚拟化适用场景

1. **依赖服务不可用**：第三方服务、跨团队服务未完成
2. **依赖服务成本高**：按调用计费的外部 API
3. **依赖服务难以构造**：异常场景、边界条件
4. **并行开发**：前后端/多服务并行开发时解耦
5. **性能测试**：模拟高延迟、低吞吐的下游服务

### 消费者驱动契约测试（CDC）流程

```
消费者定义期望 → 生成契约 → 共享给提供者 → 提供者验证契约 → 持续集成
```

## 动手实操

### WireMock：HTTP 服务 Stub

```java
// WireMockTest.java - Java 实现
import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.*;
import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static com.github.tomakehurst.wiremock.core.WireMockConfiguration.options;

public class PaymentServiceTest {

    private static WireMockServer wireMockServer;

    @BeforeAll
    static void setup() {
        wireMockServer = new WireMockServer(options().port=8888));
        wireMockServer.start();
        WireMock.configureFor("localhost", 8888);
    }

    @AfterAll
    static void teardown() {
        wireMockServer.stop();
    }

    @AfterEach
    void reset() {
        WireMock.resetAllRequests();
    }

    @Test
    void testPaymentSuccess() {
        // 配置 Stub
        stubFor(post(urlEqualTo("/api/v1/payments"))
            .withHeader("Content-Type", containing("application/json"))
            .withRequestBody(containing("\"amount\":99.99"))
            .willReturn(aResponse()
                .withStatus(201)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "payment_id": "pay_001",
                        "status": "success",
                        "amount": 99.99,
                        "currency": "CNY",
                        "created_at": "2025-01-15T10:30:00Z"
                    }
                    """)));

        // 执行测试
        PaymentClient client = new PaymentClient("http://localhost:8888");
        PaymentResult result = client.createPayment("ord_123", 99.99, "CNY");

        // 验证结果
        assertEquals("success", result.getStatus());
        assertEquals("pay_001", result.getPaymentId());

        // 验证请求确实被调用
        verify(postRequestedFor(urlEqualTo("/api/v1/payments"))
            .withRequestBody(containing("ord_123")));
    }

    @Test
    void testPaymentTimeout() {
        // 模拟超时场景
        stubFor(post(urlEqualTo("/api/v1/payments"))
            .willReturn(aResponse()
                .withFixedDelay(5000)
                .withStatus(200)));

        // 验证超时处理逻辑
        assertThrows(TimeoutException.class, () -> {
            PaymentClient client = new PaymentClient("http://localhost:8888", 2000);
            client.createPayment("ord_123", 99.99, "CNY");
        });
    }

    @Test
    void testPaymentServerError() {
        // 模拟服务端错误
        stubFor(post(urlEqualTo("/api/v1/payments"))
            .willReturn(aResponse()
                .withStatus(503)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"error": "service_unavailable", "message": "Payment service temporarily unavailable"}
                    """)));

        PaymentClient client = new PaymentClient("http://localhost:8888");
        PaymentResult result = client.createPayment("ord_123", 99.99, "CNY");

        assertEquals("error", result.getStatus());
    }
}
```

```python
# wiremock_test.py - Python 实现
import requests
import pytest
from wiremock.server import WireMockServer
from wiremock.constants import Config

@pytest.fixture(scope="session")
def wiremock():
    server = WireMockServer()
    server.start()
    Config.base_url = "http://localhost:5001/__admin"
    yield server
    server.stop()

class TestPaymentService:

    def test_create_payment_success(self, wiremock):
        # 配置 Stub
        mapping = {
            "request": {
                "method": "POST",
                "url": "/api/v1/payments",
                "bodyPatterns": [
                    {"contains": "\"amount\":99.99"}
                ]
            },
            "response": {
                "status": 201,
                "headers": {"Content-Type": "application/json"},
                "jsonBody": {
                    "payment_id": "pay_001",
                    "status": "success",
                    "amount": 99.99,
                    "currency": "CNY"
                }
            }
        }

        # 注册 Stub
        requests.post(
            "http://localhost:5001/__admin/mappings",
            json=mapping
        )

        # 执行测试
        response = requests.post(
            "http://localhost:5000/api/v1/payments",
            json={"order_id": "ord_123", "amount": 99.99, "currency": "CNY"}
        )

        assert response.status_code == 201
        data = response.json()
        assert data["status"] == "success"
        assert data["payment_id"] == "pay_001"

    def test_payment_insufficient_balance(self, wiremock):
        # 模拟余额不足
        mapping = {
            "request": {
                "method": "POST",
                "url": "/api/v1/payments"
            },
            "response": {
                "status": 400,
                "jsonBody": {
                    "error": "insufficient_balance",
                    "message": "账户余额不足"
                }
            }
        }
        requests.post(
            "http://localhost:5001/__admin/mappings",
            json=mapping
        )

        response = requests.post(
            "http://localhost:5000/api/v1/payments",
            json={"order_id": "ord_123", "amount": 99999, "currency": "CNY"}
        )

        assert response.status_code == 400
        assert response.json()["error"] == "insufficient_balance"
```

```json
// wiremock_mappings/payment-success.json - 独立映射文件
{
  "mappings": [
    {
      "name": "payment-success",
      "request": {
        "method": "POST",
        "urlPath": "/api/v1/payments",
        "bodyPatterns": [
          {
            "matchesJsonPath": "$.amount",
            "matches": "\\d+\\.\\d{2}"
          }
        ]
      },
      "response": {
        "status": 201,
        "headers": {
          "Content-Type": "application/json"
        },
        "jsonBody": {
          "payment_id": "pay_${random.uuid}",
          "status": "success",
          "amount": "{{jsonPath request.body '$.amount'}}",
          "created_at": "{{now format='yyyy-MM-dd'T'HH:mm:ss'Z'}}"
        },
        "transformers": ["response-template"]
      }
    },
    {
      "name": "payment-not-found",
      "request": {
        "method": "GET",
        "urlPathPattern": "/api/v1/payments/.*"
      },
      "response": {
        "status": 404,
        "jsonBody": {
          "error": "payment_not_found"
        }
      }
    }
  ]
}
```

### Mountebank：多协议服务虚拟化

```bash
# 安装 Mountebank
npm install -g mountebank

# 启动 Mountebank
mb start --port 2525
```

```json
// imposters.json - 多协议服务虚拟化配置
{
  "port": 4545,
  "protocol": "http",
  "name": "Payment Service Virtual",
  "stubs": [
    {
      "predicates": [
        {
          "equals": {
            "method": "POST",
            "path": "/api/v1/payments"
          }
        },
        {
          "jsonpath": {
            "selector": "$.amount"
          },
          "is": {
            "jsonpath": {
              "selector": "$.amount"
            }
          }
        }
      ],
      "responses": [
        {
          "is": {
            "statusCode": 201,
            "headers": {
              "Content-Type": "application/json"
            },
            "body": {
              "payment_id": "pay_{{random.uuid}}",
              "status": "success",
              "amount": "{{request.body.amount}}",
              "currency": "{{request.body.currency}}"
            }
          },
          "behaviors": [
            {
              "wait": 100
            }
          ]
        }
      ]
    },
    {
      "predicates": [
        {
          "equals": {
            "method": "GET",
            "path": "/api/v1/payments/{{request.path.[3]}}"
          }
        }
      ],
      "responses": [
        {
          "is": {
            "statusCode": 200,
            "body": {
              "payment_id": "{{request.path.[3]}}",
              "status": "completed"
            }
          }
        }
      ]
    }
  ]
}
```

```json
// imposters-tcp.json - TCP 协议虚拟化
{
  "port": 4546,
  "protocol": "tcp",
  "name": "Legacy Payment TCP Service",
  "mode": "text",
  "stubs": [
    {
      "predicates": [
        {
          "contains": "PAYMENT_REQ"
        }
      ],
      "responses": [
        {
          "is": {
            "data": "PAYMENT_RESP|{{random.uuid}}|SUCCESS|00\r\n"
          }
        }
      ]
    },
    {
      "predicates": [
        {
          "contains": "STATUS_CHECK"
        }
      ],
      "responses": [
        {
          "is": {
            "data": "STATUS_RESP|OK|ACTIVE\r\n"
          }
        }
      ]
    }
  ]
}
```

```bash
# 通过 API 动态创建 Imposter
curl -X POST http://localhost:2525/imposters \
  -H "Content-Type: application/json" \
  -d @imposters.json

# 查看所有 Imposters
curl http://localhost:2525/imposters

# 删除 Imposter
curl -X DELETE http://localhost:2525/imposters/4545

# 查看请求记录
curl http://localhost:2525/imposters/4545/stubs
```

### 消费者驱动契约测试：Pact 完整实现

```python
# test_order_contract.py - 消费者端契约定义
import pytest
from pact import Consumer, Provider
import requests

@pytest.fixture(scope="module")
def pact():
    pact = Consumer('OrderService').has_pact_with(
        Provider('PaymentService'),
        host_name='localhost',
        port=1234,
        pact_dir='./pacts',
        log_dir='./logs'
    )
    pact.start_service()
    yield pact
    pact.stop_service()

class TestCreatePaymentContract:

    def test_create_payment_returns_201(self, pact):
        expected_body = {
            'payment_id': 'pay_123456',
            'status': 'pending',
            'amount': 99.99,
            'currency': 'CNY',
            'created_at': '2025-01-15T10:30:00Z'
        }

        (pact
         .given('payment service is available')
         .upon_receiving('a request to create a payment for a valid order')
         .with_request(
             method='POST',
             path='/api/v1/payments',
             body={
                 'order_id': 'ord_789',
                 'amount': 99.99,
                 'currency': 'CNY'
             },
             headers={'Content-Type': 'application/json'}
         )
         .will_respond_with(
             status=201,
             body=expected_body,
             headers={'Content-Type': 'application/json'}
         ))

        with pact:
            response = requests.post(
                f"{pact.uri}/api/v1/payments",
                json={
                    'order_id': 'ord_789',
                    'amount': 99.99,
                    'currency': 'CNY'
                }
            )
            assert response.status_code == 201
            assert response.json()['status'] == 'pending'

    def test_create_payment_invalid_amount_returns_400(self, pact):
        (pact
         .given('payment service is available')
         .upon_receiving('a request to create a payment with negative amount')
         .with_request(
             method='POST',
             path='/api/v1/payments',
             body={
                 'order_id': 'ord_789',
                 'amount': -10.00,
                 'currency': 'CNY'
             },
             headers={'Content-Type': 'application/json'}
         )
         .will_respond_with(
             status=400,
             body={
                 'error': 'invalid_amount',
                 'message': 'Amount must be positive'
             }
         ))

        with pact:
            response = requests.post(
                f"{pact.uri}/api/v1/payments",
                json={
                    'order_id': 'ord_789',
                    'amount': -10.00,
                    'currency': 'CNY'
                }
            )
            assert response.status_code == 400

    def test_get_payment_by_id(self, pact):
        (pact
         .given('a payment with id pay_123456 exists')
         .upon_receiving('a request to get payment by id')
         .with_request(
             method='GET',
             path='/api/v1/payments/pay_123456'
         )
         .will_respond_with(
             status=200,
             body={
                 'payment_id': 'pay_123456',
                 'status': 'completed',
                 'amount': 99.99,
                 'currency': 'CNY'
             }
         ))

        with pact:
            response = requests.get(
                f"{pact.uri}/api/v1/payments/pay_123456"
            )
            assert response.status_code == 200
            assert response.json()['payment_id'] == 'pay_123456'
```

```python
# test_payment_provider_verify.py - 提供者端契约验证
import pytest
from pact import Verifier
import os

def test_verify_pact_contracts():
    """验证 PaymentService 满足所有消费者定义的契约"""

    verifier = Verifier(
        provider='PaymentService',
        provider_base_url=os.environ.get('PAYMENT_SERVICE_URL', 'http://localhost:8080')
    )

    # 从 Pact Broker 获取并验证契约
    success, logs = verifier.verify_with_broker(
        broker_url='https://pact-broker.internal.com',
        broker_token=os.environ['PACT_BROKER_TOKEN'],
        provider_app_version='2.1.0',
        publish_verification_results=True
    )

    assert success, f"契约验证失败: {logs}"
```

### ShopTest 完整 Mock 示例

```python
# shoptest_mocks.py - ShopTest 项目完整 Mock 配置
import pytest
import responses
import requests
from unittest.mock import patch, MagicMock
from wiremock.server import WireMockServer

class ShopTestMocks:
    """ShopTest 项目 Mock 管理类"""

    def __init__(self):
        self.wiremock = None
        self.base_url = "http://localhost:5001"

    def start(self):
        self.wiremock = WireMockServer(port=5001)
        self.wiremock.start()

    def stop(self):
        if self.wiremock:
            self.wiremock.stop()

    def setup_payment_service(self):
        """配置支付服务 Mock"""
        mappings = [
            {
                "request": {
                    "method": "POST",
                    "url": "/api/v1/payments"
                },
                "response": {
                    "status": 201,
                    "jsonBody": {
                        "payment_id": "pay_mock_001",
                        "status": "success",
                        "amount": 199.99,
                        "currency": "CNY"
                    }
                }
            },
            {
                "request": {
                    "method": "GET",
                    "urlPattern": "/api/v1/payments/.*"
                },
                "response": {
                    "status": 200,
                    "jsonBody": {
                        "status": "completed"
                    }
                }
            },
            {
                "request": {
                    "method": "POST",
                    "url": "/api/v1/payments/refund"
                },
                "response": {
                    "status": 200,
                    "jsonBody": {
                        "refund_id": "ref_mock_001",
                        "status": "refunded"
                    }
                }
            }
        ]
        for mapping in mappings:
            requests.post(f"{self.base_url}/__admin/mappings", json=mapping)

    def setup_inventory_service(self):
        """配置库存服务 Mock"""
        mappings = [
            {
                "request": {
                    "method": "GET",
                    "urlPattern": "/api/v1/inventory/.*"
                },
                "response": {
                    "status": 200,
                    "jsonBody": {
                        "product_id": "prod_001",
                        "available": True,
                        "quantity": 100
                    }
                }
            },
            {
                "request": {
                    "method": "POST",
                    "url": "/api/v1/inventory/reserve"
                },
                "response": {
                    "status": 200,
                    "jsonBody": {
                        "reservation_id": "res_mock_001",
                        "status": "reserved"
                    }
                }
            },
            {
                "request": {
                    "method": "POST",
                    "url": "/api/v1/inventory/release"
                },
                "response": {
                    "status": 200,
                    "jsonBody": {
                        "status": "released"
                    }
                }
            }
        ]
        for mapping in mappings:
            requests.post(f"{self.base_url}/__admin/mappings", json=mapping)

    def setup_user_service(self):
        """配置用户服务 Mock"""
        mappings = [
            {
                "request": {
                    "method": "GET",
                    "urlPattern": "/api/v1/users/.*"
                },
                "response": {
                    "status": 200,
                    "jsonBody": {
                        "user_id": "user_001",
                        "name": "测试用户",
                        "email": "test@shoptest.com",
                        "vip_level": "gold"
                    }
                }
            },
            {
                "request": {
                    "method": "GET",
                    "urlPattern": "/api/v1/users/.*/addresses"
                },
                "response": {
                    "status": 200,
                    "jsonBody": [
                        {
                            "address_id": "addr_001",
                            "province": "广东省",
                            "city": "深圳市",
                            "district": "南山区",
                            "detail": "科技园路1号",
                            "is_default": True
                        }
                    ]
                }
            }
        ]
        for mapping in mappings:
            requests.post(f"{self.base_url}/__admin/mappings", json=mapping)


# pytest fixtures
@pytest.fixture(scope="session")
def shoptest_mocks():
    mocks = ShopTestMocks()
    mocks.start()
    mocks.setup_payment_service()
    mocks.setup_inventory_service()
    mocks.setup_user_service()
    yield mocks
    mocks.stop()


# 测试用例
class TestOrderFlow:

    def test_create_order_with_mock_services(self, shoptest_mocks):
        """使用 Mock 服务测试完整下单流程"""
        base_url = shoptest_mocks.base_url

        # 1. 查询用户信息
        user_resp = requests.get(f"{base_url}/api/v1/users/user_001")
        assert user_resp.status_code == 200
        user = user_resp.json()
        assert user["vip_level"] == "gold"

        # 2. 查询库存
        inv_resp = requests.get(f"{base_url}/api/v1/inventory/prod_001")
        assert inv_resp.status_code == 200
        assert inv_resp.json()["available"] is True

        # 3. 创建支付
        pay_resp = requests.post(f"{base_url}/api/v1/payments", json={
            "order_id": "ord_test_001",
            "amount": 199.99,
            "currency": "CNY"
        })
        assert pay_resp.status_code == 201
        assert pay_resp.json()["status"] == "success"

        # 4. 预留库存
        reserve_resp = requests.post(f"{base_url}/api/v1/inventory/reserve", json={
            "product_id": "prod_001",
            "quantity": 1
        })
        assert reserve_resp.status_code == 200

    def test_refund_flow_with_mock(self, shoptest_mocks):
        """测试退款流程"""
        base_url = shoptest_mocks.base_url

        refund_resp = requests.post(f"{base_url}/api/v1/payments/refund", json={
            "payment_id": "pay_mock_001",
            "reason": "用户取消"
        })
        assert refund_resp.status_code == 200
        assert refund_resp.json()["status"] == "refunded"
```

```python
# responses_mock_example.py - 使用 responses 库进行轻量级 Mock
import responses
import requests
import pytest

@responses.activate
def test_order_creation_with_responses_mock():
    """使用 responses 库 Mock HTTP 调用"""

    # Mock 支付服务
    responses.add(
        responses.POST,
        "https://payment-service.internal.com/api/v1/payments",
        json={
            "payment_id": "pay_001",
            "status": "success",
            "amount": 199.99
        },
        status=201
    )

    # Mock 库存服务
    responses.add(
        responses.GET,
        "https://inventory-service.internal.com/api/v1/inventory/prod_001",
        json={"available": True, "quantity": 50},
        status=200
    )

    # Mock 库存预留
    responses.add(
        responses.POST,
        "https://inventory-service.internal.com/api/v1/inventory/reserve",
        json={"reservation_id": "res_001", "status": "reserved"},
        status=200
    )

    # 执行测试
    payment_resp = requests.post(
        "https://payment-service.internal.com/api/v1/payments",
        json={"order_id": "ord_001", "amount": 199.99}
    )
    assert payment_resp.status_code == 201

    inventory_resp = requests.get(
        "https://inventory-service.internal.com/api/v1/inventory/prod_001"
    )
    assert inventory_resp.json()["available"] is True

    # 验证调用次数
    assert len(responses.calls) == 2
    assert responses.calls[0].request.url == "https://payment-service.internal.com/api/v1/payments"
```

## 常见坑

1. **Stub 与实际服务行为不一致**：Mock 返回的数据格式与真实服务差异导致集成时暴露问题。解决方案：定期对比 Stub 与真实服务的响应，使用契约测试保证一致性
2. **WireMock 端口冲突**：多个测试套件同时运行导致端口占用。解决方案：使用随机端口或测试套件间端口分配策略
3. **契约测试维护成本高**：服务变更时契约未同步更新。解决方案：将契约验证纳入 CI/CD 流水线，强制提供者端验证
4. **Mock 覆盖不全**：仅 Mock 了正常路径，未覆盖异常场景。解决方案：根据测试用例设计完整的 Stub 矩阵（正常/异常/边界）
5. **Mountebank Imposter 未清理**：测试结束后 Imposter 残留影响后续测试。解决方案：在 teardown 中调用 DELETE API 清理

## 自测清单

- [ ] 能区分服务虚拟化、Mock、Stub 的差异
- [ ] 能使用 WireMock 配置 HTTP Stub（Java 或 Python）
- [ ] 能使用 Mountebank 配置多协议虚拟化
- [ ] 能编写消费者端 Pact 契约测试
- [ ] 能配置提供者端契约验证
- [ ] 能在 ShopTest 项目中设计完整的 Mock 策略

## 延伸阅读

- [WireMock 官方文档](http://wiremock.org/docs/)
- [Mountebank 官方文档](http://www.mbtest.org/)
- [Pact 官方文档](https://docs.pact.io/)
- [Pact Broker 文档](https://docs.pact.io/pact_broker)
- [Service Virtualization - Martin Fowler](https://martinfowler.com/articles/mocksArentStubs.html)
- [responses 库文档](https://github.com/getsentry/responses)
