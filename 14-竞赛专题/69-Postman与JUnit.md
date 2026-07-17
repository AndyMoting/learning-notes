# 69-Postman与JUnit
> 课时：45 min | 难度：★★★

## 学习目标
- 掌握 Postman 数据驱动与 Token 关联的完整流程
- 理解 JUnit 语句覆盖率的实现方法
- 学会参数化测试的两种模式（Parameterized / 手动数据驱动）

## 核心概念

### Postman 数据驱动

**流程：** Collection → Variables → JSON 数据文件 → Collection Runner

**步骤：**
1. 创建 Collection，新建 Request
2. 在请求中使用 `{{变量名}}` 引用变量
3. 准备 JSON 数据文件（格式见下）
4. Collection Runner 中选择 JSON 文件，设置 Iterations
5. 运行并查看结果

**JSON 数据文件格式：**

```json
[
  {"username": "admin", "password": "123456", "expected": "success"},
  {"username": "admin", "password": "wrong", "expected": "fail"},
  {"username": "locked", "password": "123456", "expected": "locked"}
]
```

**请求中变量引用：**

```
POST http://api.example.com/login
Body (JSON):
{
  "username": "{{username}}",
  "password": "{{password}}"
}
```

**Tests 脚本（断言）：**

```javascript
var jsonData = pm.response.json();
pm.test("Response code is 200", function () {
    pm.response.to.have.status(200);
});
pm.test("Login result matches expected", function () {
    pm.expect(jsonData.result).to.eql(pm.iterationData.get("expected"));
});
```

### Token 关联

**流程：** 登录请求 → Tests 中设置环境变量 → 后续请求使用 `{{token}}`

**登录请求 Tests 脚本：**

```javascript
var jsonData = pm.response.json();
pm.environment.set("token", jsonData.data.token);
pm.test("Token saved", function () {
    pm.expect(jsonData.data.token).to.not.be.undefined;
});
```

**后续请求 Header：**

```
Authorization: Bearer {{token}}
Content-Type: application/json
```

### Postman 截图要求

| 题号 | 截图内容 |
|------|---------|
| 数据驱动 | Body（含变量）、Tests（断言代码）、Runner 结果 |
| Token 关联 | Authorization 配置、Tests（设变量代码）、后续请求 Header |
| 结果验证 | Response Body、Test Results 面板 |

### JUnit 语句覆盖率

**流程：** 代码分析 → 确定覆盖目标 → 设计最少测试数据 → 编写 @Test 方法

**覆盖率计算：**

```
语句覆盖率 = (已执行语句数 / 总可执行语句数) × 100%
```

**分析步骤：**
1. 阅读被测方法，标记所有可执行语句
2. 识别分支结构（if/else、switch、循环）
3. 设计测试数据使每条语句至少执行一次
4. 编写测试方法验证

**示例被测代码：**

```java
public class Calculator {
    public int divide(int a, int b) {
        if (b == 0) {          // 语句 1
            throw new ArithmeticException("除数不能为0");
        }
        int result = a / b;    // 语句 2
        return result;         // 语句 3
    }
}
```

**测试数据设计：**

| 用例 | a | b | 覆盖语句 |
|------|---|---|---------|
| 1 | 10 | 2 | 语句 2、3 |
| 2 | 10 | 0 | 语句 1 |

**JUnit 测试类：**

```java
import org.junit.Test;
import static org.junit.Assert.*;

public class CalculatorTest {

    @Test
    public void testDivideNormal() {
        Calculator calc = new Calculator();
        int result = calc.divide(10, 2);
        assertEquals(5, result);
    }

    @Test(expected = ArithmeticException.class)
    public void testDivideByZero() {
        Calculator calc = new Calculator();
        calc.divide(10, 0);
    }
}
```

### Parameterized 参数化模式

```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.Parameterized;
import org.junit.runners.Parameterized.Parameters;
import java.util.Arrays;
import java.util.Collection;

@RunWith(Parameterized.class)
public class CalculatorParameterizedTest {

    private int a;
    private int b;
    private int expected;

    public CalculatorParameterizedTest(int a, int b, int expected) {
        this.a = a;
        this.b = b;
        this.expected = expected;
    }

    @Parameters
    public static Collection<Object[]> data() {
        return Arrays.asList(new Object[][] {
            {10, 2, 5},
            {20, 4, 5},
            {15, 3, 5},
            {100, 10, 10}
        });
    }

    @Test
    public void testDivide() {
        Calculator calc = new Calculator();
        assertEquals(expected, calc.divide(a, b));
    }
}
```

## 动手实操

1. 在 Postman 中创建 Collection，完成数据驱动与 Token 关联各一题
2. 对给定 Java 方法进行语句覆盖率分析，设计测试数据
3. 编写 JUnit 测试类，使用 Parameterized 模式
4. 使用 JaCoCo 或 IDE 内置工具验证覆盖率达标

## 常见坑
- Postman JSON 数据文件格式错误——必须是数组，字段名与变量名一致
- Token 关联中环境变量名拼写错误——大小写敏感
- JUnit 测试方法未加 @Test 注解——方法不执行
- Parameterized 构造方法参数顺序与 @Parameters 数据不匹配——运行时异常
- 语句覆盖率分析遗漏异常分支——catch 块中的语句也需覆盖
- Postman 截图缺少 Test Results 面板——占分点

## 自测清单
- [ ] 能写出 Postman JSON 数据文件格式
- [ ] 能编写 Token 关联的 Tests 脚本
- [ ] 能列出 Postman 截图要求
- [ ] 能计算语句覆盖率
- [ ] 能编写 Parameterized 测试类
- [ ] 能设计覆盖所有语句的最少测试数据

## 延伸阅读
- Postman 官方文档：Collection Runner
- JUnit 4 Parameterized 指南
- JaCoCo 覆盖率工具使用
- 语句覆盖率与分支覆盖率对比
