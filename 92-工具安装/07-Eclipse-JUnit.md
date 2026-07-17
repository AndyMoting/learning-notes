# 07-Eclipse-JUnit
> 课时：45 min | 难度：★★★

## 安装步骤

### 安装 Eclipse IDE for Java Developers

**所有平台：**

1. 访问 <https://www.eclipse.org/downloads/packages/>
2. 下载 "Eclipse IDE for Java Developers"
3. 解压/安装：

**Windows：** 运行 `eclipse-inst-jre-win64.exe`，选择 "Eclipse IDE for Java Developers"

**macOS：** 将 Eclipse 拖入 `Applications`

**Linux：** 解压至 `/opt/eclipse`，创建桌面快捷方式：

```bash
sudo tar -xzf eclipse-java-*.tar.gz -C /opt/
cat > ~/.local/share/applications/eclipse.desktop << EOF
[Desktop Entry]
Name=Eclipse
Type=Application
Exec=/opt/eclipse/eclipse
Icon=/opt/eclipse/icon.xpm
Categories=Development;
EOF
```

4. 启动 Eclipse，选择 Workspace 路径（如 `C:\workspace` 或 `~/workspace`）
5. 首次启动后关闭 Welcome 页面

### 配置 JDK

1. `Window → Preferences → Java → Installed JREs`
2. 点击 `Add → Standard VM`
3. JRE home 选择 JDK 安装目录（如 `C:\Program Files\Eclipse Adoptium\jdk-17`）
4. 点击 `Finish`，勾选设为默认

### 配置 JUnit 5

1. 创建 Java 项目：`File → New → Java Project`
2. 命名：`JUnit-Demo`，点击 `Finish`
3. 右键项目 → `Build Path → Add Libraries → JUnit → JUnit 5 → Finish`

或手动添加（Maven 项目）：

右键项目 → `Configure → Convert to Maven Project`，在 `pom.xml` 中添加：

```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

右键项目 → `Maven → Update Project`。

### 创建第一个测试类

1. 右键 `src/test/java` → `New → JUnit Test Case`
2. 命名：`CalculatorTest.java`
3. 输入以下代码：

```java
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;

class CalculatorTest {

    private Calculator calc;

    @BeforeEach
    void setUp() {
        calc = new Calculator();
    }

    @Test
    void testAdd() {
        assertEquals(5, calc.add(2, 3));
    }

    @Test
    void testDivideByZero() {
        assertThrows(ArithmeticException.class, () -> calc.divide(1, 0));
    }
}
```

4. 创建 `Calculator.java`：

```java
public class Calculator {
    public int add(int a, int b) { return a + b; }
    public int divide(int a, int b) { return a / b; }
}
```

5. 右键 `CalculatorTest.java` → `Run As → JUnit Test`
6. JUnit 视图显示绿色进度条，`2 tests passed`

### 创建参数化测试

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorParameterizedTest {

    private Calculator calc = new Calculator();

    @ParameterizedTest
    @CsvSource({
        "1, 1, 2",
        "2, 3, 5",
        "10, 20, 30",
        "-1, 1, 0"
    })
    void testAddWithParameters(int a, int b, int expected) {
        assertEquals(expected, calc.add(a, b));
    }
}
```

运行后 JUnit 视图显示 4 条测试记录，每条独立通过或失败。

### 安装 EclEmma 覆盖率插件

1. `Help → Eclipse Marketplace`
2. 搜索 "EclEmma"，点击 `Install`
3. 确认选择，接受协议，完成安装
4. 重启 Eclipse

### 查看覆盖率

1. 右键测试类 → `Coverage As → JUnit Test`
2. 打开 `Window → Show View → Java → Coverage`
3. Coverage 视图显示覆盖率百分比
4. 编辑器中代码行颜色：
   - 绿色：已覆盖
   - 红色：未覆盖
   - 黄色：部分覆盖（分支）

## 验证安装

创建测试类并运行，预期 JUnit 视图输出：

```
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
```

Coverage 视图显示覆盖率百分比（如 `85.3%`）。

## 常见坑

1. **JUnit 5 无法识别** — 确认项目 Build Path 中 JUnit Library 版本为 5；Maven 项目检查 `pom.xml` 中 `junit-jupiter` 依赖
2. **Eclipse 启动报错 "Java was started but returned exit code=13"** — JDK 版本与 Eclipse 位数不匹配（32/64位），确保 Eclipse 和 JDK 同为 64 位
3. **@Test 注解报错** — 导入错误，确认使用 `org.junit.jupiter.api.Test`（JUnit 5）而非 `org.junit.Test`（JUnit 4）
4. **覆盖率视图为空** — 使用 `Coverage As` 而非 `Run As` 执行测试；确认 EclEmma 已正确安装
5. **中文乱码** — `Window → Preferences → General → Workspace → Text file encoding` 设置为 `UTF-8`

## 延伸阅读

- <https://www.eclipse.org/downloads/packages/>
- <https://junit.org/junit5/docs/current/user-guide/>
- <https://eclemma.org/>
- <https://www.vogella.com/tutorials/JUnit/article.html>
- <https://www.vogella.com/tutorials/Eclipse/article.html>
