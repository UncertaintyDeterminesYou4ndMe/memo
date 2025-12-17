```markdown
# macOS 上查看与管理多个 Java/JDK 版本

本文档适合作为日常查阅的备忘录，主要包括：

- 如何查看当前 Java/JDK 版本
- 如何查看系统中安装了哪些 JDK
- 如何在 macOS 上切换和维护多个 Java 版本（系统自带工具 / jenv / SDKMAN）

---

## 一、查看当前 Java / JDK 信息

### 1. 查看当前 `java` 版本

```bash
java -version
```

示例输出：

```text
java version "11.0.28" 2024-10-15 LTS
Java(TM) SE Runtime Environment ...
Java HotSpot(TM) 64-Bit Server VM ...
```

### 2. 查看当前 `javac`（Java 编译器）版本

```bash
javac -version
```

示例：

```text
javac 11.0.28
```

### 3. 查看当前 `JAVA_HOME`

```bash
echo $JAVA_HOME
```

示例输出：

```text
/Library/Java/JavaVirtualMachines/jdk-11.jdk/Contents/Home
```

如果是空的，说明当前 shell 中没有设置 `JAVA_HOME`。

---

## 二、查看系统中安装的所有 JDK

macOS 自带一个工具：`/usr/libexec/java_home`。

### 1. 列出所有 JDK

```bash
/usr/libexec/java_home -V
```

注意是大写 `-V`。

示例输出（真实环境示例）：

```text
Matching Java Virtual Machines (3):
    11.0.28 (arm64) "Oracle Corporation" - "Java SE 11.0.28" /Library/Java/JavaVirtualMachines/jdk-11.jdk/Contents/Home
    1.8.441.07 (arm64) "Oracle Corporation" - "Java" /Library/Internet Plug-Ins/JavaAppletPlugin.plugin/Contents/Home
    1.8.0_441 (arm64) "Oracle Corporation" - "Java SE 8" /Library/Java/JavaVirtualMachines/jdk-1.8.jdk/Contents/Home
/Library/Java/JavaVirtualMachines/jdk-11.jdk/Contents/Home
```

说明：

- 上面列出了 3 个 Java 运行环境：
  - 正经 JDK 11：`/Library/Java/JavaVirtualMachines/jdk-11.jdk/...`
  - 浏览器 Java 插件（JavaAppletPlugin）：可忽略
  - 正经 JDK 8：`/Library/Java/JavaVirtualMachines/jdk-1.8.jdk/...`
- 最后一行是当前默认 `JAVA_HOME`（这里是 JDK 11）。

### 2. 获取指定版本的 `JAVA_HOME`

```bash
/usr/libexec/java_home -v 11
/usr/libexec/java_home -v 1.8
```

---

## 三、使用系统自带 `/usr/libexec/java_home` 管理多个版本（推荐基础用法）

### 3.1 临时切换 Java 版本（仅当前终端）

#### 切换到 Java 8

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 1.8)
export PATH=$JAVA_HOME/bin:$PATH

java -version
echo $JAVA_HOME
```

#### 切换到 Java 11

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 11)
export PATH=$JAVA_HOME/bin:$PATH

java -version
```

> 以上修改仅对当前终端有效，关闭终端窗口后失效。

---

### 3.2 设置默认 Java 版本（持久生效）

macOS 默认 shell 是 zsh，编辑 `~/.zshrc`：

```bash
vim ~/.zshrc
```

#### 默认使用 Java 11

```bash
# 默认使用 Java 11
export JAVA_HOME=$(/usr/libexec/java_home -v 11)
export PATH=$JAVA_HOME/bin:$PATH
```

#### 默认使用 Java 8

```bash
# 默认使用 Java 8
export JAVA_HOME=$(/usr/libexec/java_home -v 1.8)
export PATH=$JAVA_HOME/bin:$PATH
```

修改后让配置立即生效：

```bash
source ~/.zshrc
```

---

### 3.3 定义一个快捷切换函数

如果需要经常切换 Java 8 / 11 等版本，可以在 `~/.zshrc` 中加入：

```bash
# jv 8 / jv 11 这样的方式切换 Java 版本
function jv() {
  export JAVA_HOME=$(/usr/libexec/java_home -v "$1")
  export PATH=$JAVA_HOME/bin:$PATH
  echo "JAVA_HOME -> $JAVA_HOME"
  java -version
}
```

使用示例：

```bash
jv 1.8   # 切到 Java 8
jv 11    # 切到 Java 11
jv 17    # 如果系统安装了 17，也可以这样切
```

---

## 四、使用 jenv 管理多个 Java 版本（支持按项目切换）

适合：有多个项目，项目之间需要使用不同 JDK（例如某项目必须用 8，另一个用 11）。

### 4.1 安装 jenv（通过 Homebrew）

```bash
brew install jenv
```

安装后在 `~/.zshrc` 中加入：

```bash
export PATH="$HOME/.jenv/bin:$PATH"
eval "$(jenv init -)"
```

然后：

```bash
source ~/.zshrc
```

### 4.2 将已有 JDK 加入 jenv

先用系统工具查看所有 JDK：

```bash
/usr/libexec/java_home -V
```

假设输出中有：

```text
/Library/Java/JavaVirtualMachines/jdk-11.jdk/Contents/Home
/Library/Java/JavaVirtualMachines/jdk-1.8.jdk/Contents/Home
```

把它们加入 jenv 管理：

```bash
jenv add /Library/Java/JavaVirtualMachines/jdk-11.jdk/Contents/Home
jenv add /Library/Java/JavaVirtualMachines/jdk-1.8.jdk/Contents/Home
```

查看 jenv 已知的 Java 版本：

```bash
jenv versions
```

示例输出：

```text
  system
  1.8
  1.8.0.441
  11
* 11.0.28
```

星号 `*` 表示当前 jenv 使用的版本。

### 4.3 使用 jenv 切换 Java 版本

#### 全局默认版本（适用于所有终端）

```bash
jenv global 11
```

#### 当前终端使用某个版本

```bash
jenv shell 1.8
```

只对当前 shell 会话有效。

#### 针对项目目录指定 Java 版本

进入项目根目录：

```bash
cd /path/to/your/project
jenv local 1.8
```

该命令会在项目目录生成一个 `.java-version` 文件。之后每次进入该目录，jenv 会自动切换到指定版本。

---

## 五、使用 SDKMAN! 安装和管理 JDK（扩展）

适合：经常需要安装不同发行版的 JDK（OpenJDK、Temurin、Zulu 等），或需要频繁升级。

### 5.1 安装 SDKMAN!

```bash
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
```

或者把以下内容写入 `~/.zshrc`：

```bash
export SDKMAN_DIR="$HOME/.sdkman"
[[ -s "$SDKMAN_DIR/bin/sdkman-init.sh" ]] && source "$SDKMAN_DIR/bin/sdkman-init.sh"
```

### 5.2 使用 SDKMAN! 管理 Java 版本

#### 查看可用 Java 版本

```bash
sdk list java
```

#### 安装指定版本（示例：Temurin 17）

```bash
sdk install java 17.0.11-tem
```

#### 切换版本

- 当前终端使用：

  ```bash
  sdk use java 17.0.11-tem
  ```

- 设置为默认：

  ```bash
  sdk default java 17.0.11-tem
  ```

---

## 六、卸载不需要的 JDK（可选）

列出已安装的 JDK：

```bash
ls /Library/Java/JavaVirtualMachines
```

删除不需要的版本（危险操作，谨慎使用）：

```bash
sudo rm -rf /Library/Java/JavaVirtualMachines/jdk-11.jdk
```

删除后注意：

- 如果使用 jenv，需要更新：`jenv versions` 检查，多余的版本可以移除。
- 确保 `~/.zshrc` 或项目配置中没有指向已删除的 JDK 路径。

---

## 七、常用命令速查表

```bash
# 查看当前 java / javac 版本
java -version
javac -version

# 查看当前 JAVA_HOME
echo $JAVA_HOME

# 查看所有已安装 JDK
/usr/libexec/java_home -V

# 获取指定版本的 JAVA_HOME
/usr/libexec/java_home -v 1.8
/usr/libexec/java_home -v 11

# 临时切换 JDK
export JAVA_HOME=$(/usr/libexec/java_home -v 11)
export PATH=$JAVA_HOME/bin:$PATH

# jenv 常用命令
jenv versions         # 列出 jenv 已管理的版本
jenv global 11        # 设为全局默认
jenv local 1.8        # 针对当前目录设为 1.8
jenv shell 11         # 当前终端使用 11

# SDKMAN! 常用命令
sdk list java
sdk install java <version>
sdk use java <version>
sdk default java <version>
```

> 建议：如果需求不复杂，优先使用系统自带 `/usr/libexec/java_home` + `~/.zshrc` 配置；项目多、要求不同版本时，再考虑上 jenv 或 SDKMAN!。
```
