# 58-API测试自动化

> 课时：45 min | 难度：★★★★☆

## 学习目标

- 理解 REST API 测试的分层策略
- 能够使用 httpx/requests 编写 API 测试
- 掌握契约测试（Contract Testing）
- 理解 API 测试与 UI 测试的协作关系

## 核心概念

### API 测试分层

```
┌─────────────────────────────────────────┐
│           Contract Test                  │  ← 接口契约验证（Consumer-Driven）
├─────────────────────────────────────────┤
│           Integration Test               │  ← 模块间接口验证（含数据库）
├─────────────────────────────────────────┤
│           Component Test                 │  ← 单服务接口验证（Mock 外部依赖）
├─────────────────────────────────────────┤
│           Unit Test                      │  ← 函数级别（不涉及网络）
└─────────────────────────────────────────┘
```

### 测试策略对比

| 层级 | 速度 | 范围 | 依赖 | 数量占比 |
|------|------|------|------|---------|
| 单元测试 | 毫秒 | 单函数 | 无 | 60% |
| 组件测试 | 秒级 | 单服务 | Mock 外部 | 20% |
| 集成测试 | 秒级 | 多服务 | 真实/容器化 | 15% |
| 契约测试 | 秒级 | 接口兼容 | Pact Broker | 5% |

## 动手实操

### 实操1：HTTP 客户端封装

```python
# api/base_client.py
"""API 测试客户端基类"""

import httpx
import logging
from typing import Dict, Any, Optional
from dataclasses import dataclass
from contextlib import contextmanager
import json

logger = logging.getLogger(__name__)


@dataclass
class APIResponse:
    """统一 API 响应封装"""
    status_code: int
    body: Any
    headers: Dict[str, str]
    elapsed_ms: float
    request_method: str
    request_url: str
    
    @property
    def is_success(self) -> bool:
        return 200 <= self.status_code < 300
    
    @property
    def is_client_error(self) -> bool:
        return 400 <= self.status_code < 500
    
    @property
    def is_server_error(self) -> bool:
        return self.status_code >= 500


class BaseAPIClient:
    """API 客户端基类"""
    
    def __init__(
        self,
        base_url: str,
        timeout: int = 30,
        headers: Dict[str, str] = None,
        verify_ssl: bool = True
    ):
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        
        self.client = httpx.Client(
            base_url=self.base_url,
            timeout=self.timeout,
            headers=headers or {},
            verify=verify_ssl,
            follow_redirects=True
        )
    
    def request(
        self,
        method: str,
        path: str,
        params: Dict = None,
        json_data: Dict = None,
        data: Any = None,
        headers: Dict = None,
        files: Dict = None,
        expected_status: int = None
    ) -> APIResponse:
        """
        通用请求方法
        
        Args:
            method: HTTP 方法 (GET/POST/PUT/PATCH/DELETE)
            path: 接口路径
            params: URL 查询参数
            json_data: JSON 请求体
            data: 表单数据
            headers: 额外请求头
            files: 上传文件
            expected_status: 期望状态码（用于自动断言）
        """
        full_url = f"{self.base_url}/{path.lstrip('/')}"
        request_id = id(params) + id(json_data)  # 简化请求追踪
        
        # 记录请求
        logger.info(
            f"[REQ-{request_id}] {method} {full_url} | "
            f"params={params} | body={json.dumps(json_data, default=str)[:200] if json_data else None}"
        )
        
        response = self.client.request(
            method=method.upper(),
            url=path,
            params=params,
            json=json_data,
            data=data,
            headers=headers,
            files=files
        )
        
        # 解析响应体
        try:
            body = response.json()
        except Exception:
            body = response.text
        
        api_response = APIResponse(
            status_code=response.status_code,
            body=body,
            headers=dict(response.headers),
            elapsed_ms=response.elapsed.total_seconds() * 1000,
            request_method=method.upper(),
            request_url=full_url
        )
        
        # 记录响应
        logger.info(
            f"[RES-{request_id}] {api_response.status_code} | "
            f"耗时: {api_response.elapsed_ms:.0f}ms"
        )
        
        # 自动断言期望状态码
        if expected_status is not None:
            assert response.status_code == expected_status, (
                f"期望状态码 {expected_status}，实际 {response.status_code}，"
                f"响应: {json.dumps(body, ensure_ascii=False)[:500]}"
            )
        
        return api_response
    
    # 便捷方法
    def get(self, path: str, **kwargs) -> APIResponse:
        return self.request("GET", path, **kwargs)
    
    def post(self, path: str, **kwargs) -> APIResponse:
        return self.request("POST", path, **kwargs)
    
    def put(self, path: str, **kwargs) -> APIResponse:
        return self.request("PUT", path, **kwargs)
    
    def patch(self, path: str, **kwargs) -> APIResponse:
        return self.request("PATCH", path, **kwargs)
    
    def delete(self, path: str, **kwargs) -> APIResponse:
        return self.request("DELETE", path, **kwargs)
    
    def set_auth_token(self, token: str, prefix: str = "Bearer"):
        """设置认证 Token"""
        self.client.headers["Authorization"] = f"{prefix} {token}"
    
    def close(self):
        self.client.close()
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        self.close()
```

### 实操2：服务层封装（Service Layer）

```python
# api/user_service.py
"""用户服务 API 封装"""

from api.base_client import BaseAPIClient, APIResponse
from typing import Dict, List, Optional


class UserService:
    """用户相关接口封装"""
    
    def __init__(self, client: BaseAPIClient):
        self.client = client
    
    def register(self, username: str, email: str, password: str) -> APIResponse:
        """用户注册"""
        return self.client.post(
            "/api/v1/users",
            json_data={
                "username": username,
                "email": email,
                "password": password
            },
            expected_status=201
        )
    
    def login(self, email: str, password: str) -> APIResponse:
        """用户登录"""
        return self.client.post(
            "/api/v1/auth/login",
            json_data={"email": email, "password": password}
        )
    
    def get_profile(self, token: str) -> APIResponse:
        """获取用户资料"""
        self.client.set_auth_token(token)
        return self.client.get("/api/v1/users/me")
    
    def update_profile(self, token: str, **fields) -> APIResponse:
        """更新用户资料"""
        self.client.set_auth_token(token)
        return self.client.patch(
            "/api/v1/users/me",
            json_data=fields
        )
    
    def delete_user(self, user_id: str, token: str) -> APIResponse:
        """删除用户"""
        self.client.set_auth_token(token)
        return self.client.delete(f"/api/v1/users/{user_id}")
    
    def list_users(self, token: str, page: int = 1, size: int = 20) -> APIResponse:
        """获取用户列表（管理员接口）"""
        self.client.set_auth_token(token)
        return self.client.get(
            "/api/v1/users",
            params={"page": page, "size": size}
        )


# api/order_service.py
"""订单服务 API 封装"""

from api.base_client import BaseAPIClient, APIResponse
from typing import Dict, List


class OrderService:
    """订单相关接口封装"""
    
    def __init__(self, client: BaseAPIClient):
        self.client = client
    
    def create_order(
        self,
        token: str,
        items: List[Dict],
        address: Dict
    ) -> APIResponse:
        """创建订单"""
        self.client.set_auth_token(token)
        return self.client.post(
            "/api/v1/orders",
            json_data={
                "items": items,
                "address": address
            },
            expected_status=201
        )
    
    def get_order(self, token: str, order_id: str) -> APIResponse:
        """获取订单详情"""
        self.client.set_auth_token(token)
        return self.client.get(f"/api/v1/orders/{order_id}")
    
    def pay_order(self, token: str, order_id: str, payment_method: str) -> APIResponse:
        """支付订单"""
        self.client.set_auth_token(token)
        return self.client.post(
            f"/api/v1/orders/{order_id}/pay",
            json_data={"payment_method": payment_method}
        )
    
    def cancel_order(self, token: str, order_id: str, reason: str) -> APIResponse:
        """取消订单"""
        self.client.set_auth_token(token)
        return self.client.post(
            f"/api/v1/orders/{order_id}/cancel",
            json_data={"reason": reason}
        )
    
    def list_orders(self, token: str, status: str = None) -> APIResponse:
        """获取订单列表"""
        self.client.set_auth_token(token)
        params = {}
        if status:
            params["status"] = status
        return self.client.get("/api/v1/orders", params=params)
```

### 实操3：API 测试用例

```python
# tests/api/test_user_api.py
"""用户 API 测试"""

import pytest
from api.base_client import BaseAPIClient
from api.user_service import UserService
from factories.user_factory import UserFactory


class TestUserAPI:
    """用户接口测试套件"""
    
    @pytest.fixture
    def api_client(self, env_config):
        """创建 API 客户端"""
        client = BaseAPIClient(base_url=env_config.api_url)
        yield client
        client.close()
    
    @pytest.fixture
    def user_service(self, api_client):
        return UserService(api_client)
    
    @pytest.fixture
    def registered_user(self, user_service):
        """注册一个测试用户"""
        user_data = UserFactory.create()
        response = user_service.register(
            username=user_data.username,
            email=user_data.email,
            password=user_data.password
        )
        assert response.is_success
        return {
            "email": user_data.email,
            "password": user_data.password,
            "user_id": response.body["id"]
        }
    
    @pytest.fixture
    def auth_token(self, user_service, registered_user) -> str:
        """获取认证 Token"""
        response = user_service.login(
            email=registered_user["email"],
            password=registered_user["password"]
        )
        assert response.is_success
        return response.body["token"]
    
    # ============ 注册接口测试 ============
    
    def test_register_success(self, user_service):
        """正常注册"""
        user_data = UserFactory.create()
        response = user_service.register(
            username=user_data.username,
            email=user_data.email,
            password=user_data.password
        )
        
        assert response.is_success
        assert response.status_code == 201
        assert "id" in response.body
        assert response.body["username"] == user_data.username
        assert response.body["email"] == user_data.email
        assert "password" not in response.body  # 响应不应包含密码
    
    def test_register_duplicate_email(self, user_service, registered_user):
        """重复邮箱注册应失败"""
        response = user_service.register(
            username="another_user",
            email=registered_user["email"],
            password="Test@123456"
        )
        
        assert response.is_client_error
        assert response.status_code == 409
    
    @pytest.mark.parametrize("email", [
        "invalid",
        "@example.com",
        "user@",
        "user@.com",
        ""
    ])
    def test_register_invalid_email(self, user_service, email):
        """无效邮箱格式应被拒绝"""
        response = user_service.register(
            username="test_user",
            email=email,
            password="Test@123456"
        )
        assert response.status_code == 400
    
    # ============ 登录接口测试 ============
    
    def test_login_success(self, user_service, registered_user):
        """正常登录"""
        response = user_service.login(
            email=registered_user["email"],
            password=registered_user["password"]
        )
        
        assert response.is_success
        assert "token" in response.body
        assert response.body["token"]
    
    def test_login_wrong_password(self, user_service, registered_user):
        """错误密码登录失败"""
        response = user_service.login(
            email=registered_user["email"],
            password="wrong_password"
        )
        
        assert response.status_code == 401
    
    # ============ 接口契约测试 ============
    
    def test_user_profile_response_schema(self, user_service, auth_token):
        """用户资料响应结构验证"""
        response = user_service.get_profile(auth_token)
        
        assert response.is_success
        
        # 验证响应结构
        expected_fields = {"id", "username", "email", "created_at"}
        actual_fields = set(response.body.keys())
        assert expected_fields.issubset(actual_fields), (
            f"缺少字段: {expected_fields - actual_fields}"
        )
        
        # 验证字段类型
        assert isinstance(response.body["id"], (str, int))
        assert isinstance(response.body["username"], str)
        assert isinstance(response.body["email"], str)
        assert "@" in response.body["email"]
    
    def test_api_latency_sla(self, user_service, auth_token):
        """API 响应时间满足 SLA（P95 < 500ms）"""
        response = user_service.get_profile(auth_token)
        
        assert response.elapsed_ms < 500, (
            f"API 响应时间 {response.elapsed_ms:.0f}ms 超过 SLA（500ms）"
        )
```

### 实操4：契约测试

```python
# tests/contracts/test_user_contract.py
"""用户服务契约测试 - 验证接口响应结构"""

import pytest
import jsonschema
from api.base_client import BaseAPIClient
from api.user_service import UserService


# API 响应 Schema 定义
USER_PROFILE_SCHEMA = {
    "type": "object",
    "required": ["id", "username", "email", "created_at"],
    "properties": {
        "id": {"type": "string"},
        "username": {"type": "string", "minLength": 1, "maxLength": 50},
        "email": {"type": "string", "format": "email"},
        "phone": {"type": "string", "pattern": "^1[3-9]\\d{9}$"},
        "created_at": {"type": "string", "format": "date-time"},
        "is_active": {"type": "boolean"}
    },
    "additionalProperties": False
}

ORDER_RESPONSE_SCHEMA = {
    "type": "object",
    "required": ["id", "user_id", "status", "total_amount", "items"],
    "properties": {
        "id": {"type": "string"},
        "user_id": {"type": "string"},
        "status": {"enum": ["pending", "paid", "shipped", "delivered", "cancelled"]},
        "total_amount": {"type": "number", "minimum": 0},
        "items": {
            "type": "array",
            "minItems": 1,
            "items": {
                "type": "object",
                "required": ["product_id", "quantity", "price"],
                "properties": {
                    "product_id": {"type": "string"},
                    "quantity": {"type": "integer", "minimum": 1},
                    "price": {"type": "number", "minimum": 0}
                }
            }
        }
    }
}


class TestUserContract:
    """用户服务契约测试"""
    
    @pytest.fixture
    def user_service(self, api_client):
        return UserService(api_client)
    
    def test_profile_response_matches_schema(self, user_service, auth_token):
        """用户资料响应符合 Schema"""
        response = user_service.get_profile(auth_token)
        
        # jsonschema 验证
        jsonschema.validate(
            instance=response.body,
            schema=USER_PROFILE_SCHEMA
        )
    
    def test_login_response_matches_schema(self, user_service, registered_user):
        """登录响应包含必需字段"""
        response = user_service.login(
            email=registered_user["email"],
            password=registered_user["password"]
        )
        
        LOGIN_SCHEMA = {
            "type": "object",
            "required": ["token", "expires_in"],
            "properties": {
                "token": {"type": "string"},
                "expires_in": {"type": "integer", "minimum": 0}
            }
        }
        
        jsonschema.validate(instance=response.body, schema=LOGIN_SCHEMA)
    
    def test_error_response_schema(self, user_service):
        """错误响应结构统一"""
        response = user_service.login(
            email="nonexistent@test.com",
            password="wrong"
        )
        
        ERROR_SCHEMA = {
            "type": "object",
            "required": ["error", "message"],
            "properties": {
                "error": {"type": "string"},
                "message": {"type": "string"},
                "details": {"type": "object"}
            }
        }
        
        jsonschema.validate(instance=response.body, schema=ERROR_SCHEMA)
```

### 实操5：数据驱动测试

```python
# tests/api/test_user_validation.py
"""用户注册参数化测试 - 数据驱动"""

import pytest
from api.base_client import BaseAPIClient
from api.user_service import UserService


class TestUserValidation:
    """用户注册数据验证测试"""
    
    @pytest.fixture
    def user_service(self, api_client):
        return UserService(api_client)
    
    @pytest.mark.parametrize("email,password,expected_status,description", [
        # 正常情况
        ("valid@test.com", "Secure@123", 201, "正常注册"),
        
        # 邮箱格式错误
        ("invalid", "Secure@123", 400, "无效邮箱"),
        ("@test.com", "Secure@123", 400, "缺少用户名"),
        ("user@", "Secure@123", 400, "缺少域名"),
        ("", "Secure@123", 400, "空邮箱"),
        
        # 密码不符合规则
        ("valid@test.com", "123", 400, "密码过短"),
        ("valid@test.com", "password", 400, "密码无大写字母"),
        ("valid@test.com", "PASSWORD", 400, "密码无小写字母"),
        ("valid@test.com", "Password", 400, "密码无数字"),
        ("valid@test.com", "Pass1234", 400, "密码无特殊字符"),
        ("valid@test.com", "", 400, "空密码"),
        
        # 边界值
        ("a" * 245 + "@t.com", "Secure@123", 400, "邮箱过长"),
        ("valid@test.com", "Aa1!" + "x" * 120, 400, "密码过长"),
    ])
    def test_register_validation(
        self, user_service, email, password, expected_status, description
    ):
        """参数化注册验证"""
        response = user_service.register(
            username=f"test_{description}",
            email=email,
            password=password
        )
        assert response.status_code == expected_status, (
            f"[{description}] 期望 {expected_status}，实际 {response.status_code}，"
            f"email={email[:30]}..., password={password[:10]}..."
        )
    
    @pytest.mark.parametrize("field,missing_field", [
        ({"username": "test", "email": "test@test.com", "password": "Pass@123"}, "none"),
        ({"email": "test@test.com", "password": "Pass@123"}, "username"),
        ({"username": "test", "password": "Pass@123"}, "email"),
        ({"username": "test", "email": "test@test.com"}, "password"),
    ])
    def test_register_required_fields(self, user_service, field, missing_field):
        """注册必填字段验证"""
        response = user_service.register(**field)
        
        if missing_field == "none":
            assert response.is_success
        else:
            assert response.status_code == 400
            assert missing_field in str(response.body).lower()
```

## 常见坑

1. **测试依赖执行顺序**：测试 A 创建的 data 被测试 B 使用——每个测试独立准备数据
2. **硬编码测试数据**：账号、ID 写死在测试代码中——使用工厂模式动态生成
3. **忽视接口契约**：只验证状态码，不验证响应结构——使用 jsonschema 做结构校验
4. **环境依赖**：测试依赖特定环境的运行服务——使用 Mock Server 或容器化依赖
5. **测试数据污染**：测试产生的数据残留在数据库中——每次测试后清理（teardown）

## 自测清单

- [ ] 能封装通用的 HTTP 客户端并统一响应格式
- [ ] 能为服务接口编写分层测试（组件/集成/契约）
- [ ] 能使用 jsonschema 验证 API 响应结构
- [ ] 能编写参数化测试覆盖边界条件
- [ ] 能设计 API 测试的 SLA 验证（延迟/可用性）

## 延伸阅读

- [REST API Testing Best Practices](https://martinfowler.com/articles/microservice-testing/)
- [Pact - Contract Testing](https://pact.io/)
- [jsonschema Documentation](https://json-schema.org/)
- [HTTPX - Python HTTP Client](https://www.python-httpx.org/)
