## 代码评审报告

### 变更摘要
本次 `git diff` 涉及两个 GitHub Actions 工作流文件：
- `.github/workflows/main-local.yml`
- `.github/workflows/maven-maven-jar.yml`

变更内容为将 CI 环境中使用的 Java 版本从 `11` 统一升级到 `17`（均保持各自的 JDK 发行版：`temurin` 和 `adopt`）。

### 影响分析

#### 1. 兼容性
- **语言特性**：Java 17 是 LTS 版本（2021.09），与 Java 11 大部分向后兼容，但移除了部分过时 API（如 `java.corba`、`java.xml.ws` 等 Java EE 模块）。若项目代码或依赖库使用了这些模块，会引发编译或运行时错误。
- **项目依赖**：需确认框架（Spring Boot、Hibernate 等）的最低版本是否支持 Java 17。例如：
  - Spring Boot 2.5+ 开始支持 JDK 17，2.7+ 更为推荐。
  - 若项目使用 Spring Boot 2.3 或更低版本，可能需要一并升级框架。
- **Maven/Gradle 插件**：`maven-compiler-plugin`、`maven-surefire-plugin` 等需要设置 `<source>` 和 `<target>` 为 17，否则仍会以 Java 11 字节码编译，但运行时由 JDK 17 执行，可能产生警告。

#### 2. 性能与安全
- Java 17 在 GC、JIT 编译等方面有显著改进（增强的 ZGC、G1 GC 优化），可能提升应用吞吐量。
- 包含更多安全修复和增强（如密封类、更严格的访问控制），降低安全隐患。
- 需注意 JDK 17 对 `Security Manager` 的废弃（计划删除），若项目依赖它需提前规划。

#### 3. CI 环境
- 两个 workflow 均修改了 `actions/setup-java@v2` 的 `java-version`，未改动其他步骤。该 action 会自动下载对应版本 JDK，但需要确保 GitHub Actions 运行器有足够磁盘和网络。
- 未修改 `distribution`，保持原有 `temurin` 和 `adopt`，兼容性良好。

### 潜在风险与问题

1. **代码兼容性问题**：CI 中仅运行 `run`（main-local.yml）或 `mvn clean install`（main-maven-jar.yml），若未启用 `-Werror` 或引入其他代码检查，编译时可能未检测到废弃 API 的使用（例如使用 `javax.xml.bind` 等移除模块）。建议在 CI 中添加 `--release 17` 或 `-Xlint:all` 强制检查。
2. **运行时兼容性**：编译时的字节码版本（若未修改 pom.xml）仍可能为 11（默认），而 JDK 17 能运行低版本字节码，但失去新特性优势。需要同步更新构建工具配置。
3. **第三方库冲突**：某些库在 Java 17 下可能因模块化限制（如反射访问限制）而抛出 `InaccessibleObjectException`。需要在 JVM 启动参数中添加 `--add-opens` 或依赖适配版本。
4. **文档与团队通知**：没有在代码注释或 commit message 中说明升级原因，可能其他开发者不知情，导致本地环境与 CI 不一致。

### 改进建议

1. **同步更新项目构建配置**  
   检查 `pom.xml` 或 `build.gradle`，将编译选项更新为 Java 17：
   - Maven:
     ```xml
     <properties>
         <maven.compiler.source>17</maven.compiler.source>
         <maven.compiler.target>17</maven.compiler.target>
         <java.version>17</java.version>
     </properties>
     ```
   - 同时确认 `spring-boot-maven-plugin` 等插件版本兼容 Java 17。

2. **增加 CI 校验步骤**  
   在 build 之前或之后增加 `-Xlint:all` 或 `--release 17`，确保无废弃或非法 API 使用：
   ```yaml
   - name: Build with Maven
     run: mvn clean install -Dmaven.compiler.release=17
   ```
   或使用 `mvn compile -Xlint:all --batch-mode` 输出警告。

3. **检查运行时依赖**  
   执行 `mvn dependency:tree` 确认没有使用已移除的 Java EE 模块（如 `javax.xml`、`javax.ws.rs` 等）。若存在，需替换为 Jakarta EE 命名空间或添加第三方实现（如 `com.sun.xml.bind:jaxb-impl`）。

4. **更新 Docker 镜像（如果适用）**  
   如果项目使用 Docker 部署，需同步更新基础镜像为 `openjdk:17-slim`、`eclipse-temurin:17` 或 `adoptopenjdk:17-jdk`，避免运行时与构建环境 JDK 版本不一致。

5. **添加构建历史对比**  
   建议在 CI 中保留一次 Java 11 的并行构建或对比测试，确认升级后测试通过率无下降。

### 评审结论
**可批准，但需补充上述改进建议中的同步更新**。当前变更仅调整了 CI 配置，未改动源代码，属于有效的技术栈升级。但评审者应要求开发者在同一 PR 中提交对 `pom.xml`（或 `build.gradle`）、Dockerfile 等配置文件的相应修改，以确保环境一致性，并在必要时添加 `--add-opens` 等 JVM 参数。建议增加一段注释说明升级原因（如：利用 Java 17 新特性、LTS 支持、性能与安全提升）。