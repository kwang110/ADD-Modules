# 新增 Python 算法模块接入指南

> 目标：在 `QSubtopicByPython` 插件里新增一个"按钮 → 对话框 → 调用 Python 脚本 → 回显结果"的功能模块。
> 本文以已完成的 **测井曲线统计（LasStat）** 为样板，按本文步骤可以完整复现一个同样的模块。

---

## 0. 总览

一个模块要动 5 个地方：

| # | 位置 | 作用 |
|---|---|---|
| 1 | `bin_qt_release_64\python2_evn\Subtopic\<模块名>\xxx.py` | 算法脚本（不参与编译，直接放进去即可） |
| 2 | `QSubtopicByPython\XxxDialog.h` / `.cpp` | 参数输入 + 调 Python + 结果展示 |
| 3 | `QSubtopicByPython\modelTool1RibbonCategory.h` / `.cpp` | 在 Ribbon 上加按钮并连接槽 |
| 4 | `QSubtopicByPython\QSubtopicByPython.vcxproj` | 把新文件加进工程（`QtMoc` + `ClCompile`） |
| 5 | `bin_qt_release_64\resource\Subtopic\` + `subtopicByPython_i.h` | 按钮图标（可复用现成图标跳过） |

数据流：

```
Ribbon 按钮
   └─ newBtNSlot()
        └─ XxxDialog::exec()
             └─ QProcess 启动 python2_evn\Python312\python.exe
                  └─ 跑 python2_evn\Subtopic\<模块>\xxx.py --input ... --output ...
                       └─ stdout 最后一行输出 JSON
                            └─ 对话框解析并显示，脚本自己写 CSV 到输出目录
```

---

## 1. 环境准备（一次性）

### 1.1 组件清单

| 组件 | 版本 / 要求 | 作用 |
|---|---|---|
| Visual Studio 2022 | 17.x（实测 17.14.45） | 编译器 + MSBuild |
| MSVC 工具集 | v143（实测 14.44） | 工程里写死 `PlatformToolset=v143` |
| Windows SDK | 10.x（任一） | 工程写的是 `WindowsTargetPlatformVersion=10.0` |
| Qt VS Tools | 3.x（实测 3.5.0） | moc/uic 集成、Qt 版本管理 |
| Qt | **6.4.3 msvc2019_64** | 必须和主程序 `bin_qt_release_64` 里的 Qt6*.dll 一致 |

> **Qt 不能用最新版。** `QApp.exe` 链接的是 `bin_qt_release_64\Qt6Core.dll`（6.4.3），插件换 6.8/6.9 会引用运行时不存在符号，加载失败。VS 可以用最新 2022 17.x。

### 1.2 安装 Visual Studio 2022

方式一（推荐，命令行）：

```powershell
winget install --id Microsoft.VisualStudio.2022.Community --source winget `
  --accept-package-agreements --accept-source-agreements `
  --override '--wait --add Microsoft.VisualStudio.Workload.NativeDesktop --includeRecommended'
```

方式二（图形界面）：运行 VS 安装器，在"工作负载"页勾选 **使用 C++ 的桌面开发**，"Visual Studio 核心编辑器"保持默认勾选（必需组件，不能取消）。

无论哪种方式，都要在右侧"安装详细信息"里确认包含：

- **MSVC v143 - VS 2022 C++ x64/x86 生成工具**
- **Windows 11 SDK**（或 Windows 10 SDK，有其一即可）

默认安装路径：`C:\Program Files\Microsoft Visual Studio\2022\Community`

### 1.3 安装 Qt VS Tools 扩展

必须先打开 VS 才能装扩展：

1. 启动 VS 2022 → **继续但无需代码**
2. **扩展 → 管理扩展** → 左侧 **Visual Studio Marketplace**
3. 搜索 `Qt Visual Studio Tools`，发布者 **Qt Group**（不是 "The Qt Company" 以外的仿冒项）
4. 点 **Install** → 提示计划安装后点 **关闭**
5. **关闭 VS**（全部窗口），自动弹出 VSIX Installer → **Modify** → 完成
6. 重新打开 VS，菜单栏出现 **Extensions → Qt VS Tools** 即成功

### 1.4 安装 Qt 6.4.3

下载在线安装器（官方地址会变，以 `download.qt.io/official_releases/online_installers/` 目录下的文件名为准）：

```
https://download.qt.io/official_releases/online_installers/qt-online-installer-windows-x64-online.exe
```

安装步骤：

1. 运行安装器 → 登录/注册一个免费 Qt 账号
2. 安装方式选 **Custom installation**
3. 在组件页把 **Archive** 勾上（不勾看不到 6.4.3 这种老版本）
4. 展开 **Qt → 6.4.3**，只勾 **MSVC 2019 64-bit**
5. 其他一律不勾（MinGW、ARM64、Sources、Preview 里的 Qt Creator 都不要）
6. 安装目录填 **`C:\Qt`**（默认会落到 `C:\Users\<用户>\Qt`，注意改）
7. 安装（约 1 GB）

装完目录结构应为：

```
C:\Qt\6.4.3\msvc2019_64\
  bin\qmake.exe
  bin\Qt6Core.dll
  lib\Qt6Core.lib
  lib\Qt6Widgets.lib
```

### 1.5 在 VS 里注册 Qt 版本（关键）

**Extensions → Qt VS Tools → Qt Versions → Add new Qt version**

- **Name**：`QT_6.4.3_msvc2019_X64`
  > **必须一字不差。** `QSubtopicByPython.vcxproj` 里写死 `<QtInstall>QT_6.4.3_msvc2019_X64</QtInstall>`，名字对不上就会报"找不到 Qt 版本"，moc/uic 全部跳过。
- **Path**：`C:\Qt\6.4.3\msvc2019_64\bin\qmake.exe`（选到 exe 文件，不是目录）

### 1.6 验证环境

```powershell
# Qt
& 'C:\Qt\6.4.3\msvc2019_64\bin\qmake.exe' -v
# QMake version 3.1
# Using Qt version 6.4.3 in C:/Qt/6.4.3/msvc2019_64/lib

# 编译器
& 'C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\*\bin\Hostx64\x64\cl.exe' 2>&1 | Select-Object -First 1

# MSBuild
Test-Path 'C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe'
```

注册是否生效可以查注册表（有 `InstallDir` 就对了）：

```powershell
reg query 'HKCU\Software\QtProject\QtVsTools\Versions\QT_6.4.3_msvc2019_X64' /v InstallDir
```

如果 Qt Versions 对话框里只有 `DefaultQtVersion` 而没有版本项，说明只填了默认名、没真正 Add，需要重新走 1.5。

### 1.7 配置限制

> **只能用 `Release|x64`**（`Publish|x64` 映射到的也是 Release）。本项目 `Debug|x64` 没有配置头文件/库目录和输出目录，Qt 模块也只有 `core`（缺 `widgets`），编不过。
>
> 解决方案里的 `x86` 平台全部映射到 x64，是保留项，不要选。

---

## 2. 第一步：写 Python 脚本

### 2.1 放置位置

```
bin_qt_release_64\
  python2_evn\
    Python312\python.exe          ← 固定解释器
    Subtopic\
      LasStat\las_stat.py         ← 每个模块一个子目录
      <你的模块>\<你的脚本>.py
```

目录名随意，但要和 C++ 里拼的路径一致。

### 2.2 脚本约定（重要，四条）

1. **参数全部走命令行**（`argparse`），不要交互输入：
   ```
   --input <输入文件> --output <输出目录>
   ```
2. **结果文件写到 `--output` 目录**。
3. **stdout 最后一行输出一个 JSON**，C++ 端靠它取结果。
4. **出错也要输出 JSON** `{"ok": false, "error": "..."}` 并以非 0 退出码结束，不要抛裸栈。

JSON 结构（与对话框解析逻辑对应）：

```json
{
  "ok": true,
  "input": "D:\\...\\TEST-001.las",
  "rows": 301,
  "curve_count": 5,
  "depth": {"top": 1000.0, "bottom": 1300.0},
  "curves": [
    {"name": "GR", "count": 301, "min": 23.44, "max": 115.97, "mean": 73.371, "std": 25.249}
  ],
  "csv": "D:\\...\\out\\TEST-001_curve_stat.csv"
}
```

### 2.3 脚本骨架

```python
# -*- coding: utf-8 -*-
import argparse, json, os, sys

def main():
    p = argparse.ArgumentParser()
    p.add_argument("--input", required=True)
    p.add_argument("--output", required=True)
    args = p.parse_args()

    # ... 你的算法 ...
    os.makedirs(args.output, exist_ok=True)
    csv_path = os.path.join(args.output, "result.csv")
    with open(csv_path, "w", encoding="utf-8-sig", newline="") as f:
        f.write("A,B\n1,2\n")

    print("[my_algo] done", flush=True)
    print(json.dumps({"ok": True, "rows": 1, "csv": csv_path}, ensure_ascii=False), flush=True)

if __name__ == "__main__":
    try:
        main()
    except Exception as exc:
        print(json.dumps({"ok": False, "error": str(exc)}, ensure_ascii=False), flush=True)
        sys.exit(1)
```

### 2.4 先独立自测

不要一上来就在软件里点，先在命令行跑通：

```powershell
cd D:\迪拜研究院\DiBaiYanJiuYuanCodeAndRX64\bin_qt_release_64
.\python2_evn\Python312\python.exe .\python2_evn\Subtopic\<模块>\xxx.py --input <测试文件> --output .\python2_evn\Subtopic\<模块>\out
```

能打印出 `{"ok": true, ...}` 再往下走。

---

## 3. 第二步：写对话框类

### 3.1 新建 `XxxDialog.h`

```cpp
#pragma once

#include <QDialog>
#include <QProcess>
#include <QString>

class QLineEdit;
class QTextEdit;
class QPushButton;

class XxxDialog : public QDialog
{
    Q_OBJECT                       // ← 必须有，否则 moc 不生成、connect 失效

public:
    explicit XxxDialog(QWidget* parent = nullptr);
    ~XxxDialog() override;

private slots:
    void onBrowseInput();
    void onBrowseOutput();
    void onRun();
    void onStdout();
    void onFinished(int exitCode, QProcess::ExitStatus status);

private:
    void showResult(const QString& output);

    QLineEdit* m_inputEdit = nullptr;
    QLineEdit* m_outputEdit = nullptr;
    QTextEdit* m_log = nullptr;
    QPushButton* m_runBtn = nullptr;
    QProcess* m_process = nullptr;
    QString m_buffer;
};
```

### 3.2 新建 `XxxDialog.cpp`（可直接改名的模板）

```cpp
#include "XxxDialog.h"

#include <QCoreApplication>
#include <QDir>
#include <QFileDialog>
#include <QFileInfo>
#include <QGroupBox>
#include <QHBoxLayout>
#include <QJsonDocument>
#include <QJsonObject>
#include <QLabel>
#include <QLineEdit>
#include <QMessageBox>
#include <QProcessEnvironment>
#include <QPushButton>
#include <QTextEdit>
#include <QVBoxLayout>

#include "pydefine.h"      // py_version = "Python312"

XxxDialog::XxxDialog(QWidget* parent) : QDialog(parent)
{
    resize(820, 560);
    setWindowTitle(QStringLiteral("我的算法"));

    QVBoxLayout* mainLayout = new QVBoxLayout(this);

    QHBoxLayout* inputRow = new QHBoxLayout;
    inputRow->addWidget(new QLabel(QStringLiteral("输入文件")));
    m_inputEdit = new QLineEdit;
    QPushButton* browseInputBtn = new QPushButton(QStringLiteral("浏览"));
    inputRow->addWidget(m_inputEdit);
    inputRow->addWidget(browseInputBtn);
    mainLayout->addLayout(inputRow);

    QHBoxLayout* outputRow = new QHBoxLayout;
    outputRow->addWidget(new QLabel(QStringLiteral("输出目录")));
    m_outputEdit = new QLineEdit;
    QPushButton* browseOutputBtn = new QPushButton(QStringLiteral("浏览"));
    outputRow->addWidget(m_outputEdit);
    outputRow->addWidget(browseOutputBtn);
    mainLayout->addLayout(outputRow);

    QGroupBox* logGroup = new QGroupBox(QStringLiteral("运行日志"));
    QVBoxLayout* logLayout = new QVBoxLayout(logGroup);
    m_log = new QTextEdit;
    m_log->setReadOnly(true);
    logLayout->addWidget(m_log);
    mainLayout->addWidget(logGroup);

    m_runBtn = new QPushButton(QStringLiteral("运行"));
    mainLayout->addWidget(m_runBtn);

    // 默认输出目录：exe 旁边的 python2_evn/Subtopic/<模块>/out
    m_outputEdit->setText(QCoreApplication::applicationDirPath()
        + "/python2_evn/Subtopic/<模块>/out");

    connect(browseInputBtn,  &QPushButton::clicked, this, &XxxDialog::onBrowseInput);
    connect(browseOutputBtn, &QPushButton::clicked, this, &XxxDialog::onBrowseOutput);
    connect(m_runBtn,        &QPushButton::clicked, this, &XxxDialog::onRun);
}

XxxDialog::~XxxDialog()
{
    if (m_process) {
        m_process->disconnect();
        if (m_process->state() != QProcess::NotRunning) {
            m_process->kill();
            m_process->waitForFinished(3000);
        }
        delete m_process;
    }
}

void XxxDialog::onBrowseInput()
{
    const QString path = QFileDialog::getOpenFileName(this, QStringLiteral("选择输入文件"));
    if (!path.isEmpty()) m_inputEdit->setText(path);
}

void XxxDialog::onBrowseOutput()
{
    const QString dir = QFileDialog::getExistingDirectory(this, QStringLiteral("选择输出目录"),
                                                          m_outputEdit->text());
    if (!dir.isEmpty()) m_outputEdit->setText(dir);
}

void XxxDialog::onRun()
{
    // ① 三个路径都相对 exe 目录，不要写绝对路径
    const QString appDir     = QCoreApplication::applicationDirPath();
    const QString pythonExe  = appDir + "/python2_evn/" + py_version + "/python.exe";
    const QString scriptPath = appDir + "/python2_evn/Subtopic/<模块>/xxx.py";

    const QString inputFile = m_inputEdit->text().trimmed();
    const QString outputDir = m_outputEdit->text().trimmed();

    if (inputFile.isEmpty() || !QFileInfo::exists(inputFile)) {
        QMessageBox::warning(this, QStringLiteral("提示"), QStringLiteral("请选择有效的输入文件。"));
        return;
    }
    if (!QFileInfo::exists(pythonExe))  { m_log->append(QStringLiteral("找不到 Python: ") + pythonExe); return; }
    if (!QFileInfo::exists(scriptPath)) { m_log->append(QStringLiteral("找不到脚本: ") + scriptPath); return; }
    if (!QDir().mkpath(outputDir))      { m_log->append(QStringLiteral("无法创建目录: ") + outputDir); return; }

    if (m_process) { m_process->disconnect(); m_process->deleteLater(); m_process = nullptr; }
    m_process = new QProcess(this);
    m_process->setProgram(pythonExe);
    m_process->setArguments({ scriptPath, "--input", inputFile, "--output", outputDir });
    m_process->setProcessChannelMode(QProcess::MergedChannels);       // stderr 并到 stdout
    m_process->setWorkingDirectory(QFileInfo(scriptPath).absolutePath());

    QProcessEnvironment env = QProcessEnvironment::systemEnvironment();
    env.insert("PYTHONUTF8", "1");                                    // 中文日志不乱码
    m_process->setProcessEnvironment(env);

    connect(m_process, &QProcess::readyReadStandardOutput, this, &XxxDialog::onStdout);
    connect(m_process, QOverload<int, QProcess::ExitStatus>::of(&QProcess::finished),
            this, &XxxDialog::onFinished);

    m_buffer.clear();
    m_log->clear();
    m_log->append(QStringLiteral("开始执行..."));
    m_runBtn->setEnabled(false);
    m_runBtn->setText(QStringLiteral("运行中..."));
    m_process->start();
}

void XxxDialog::onStdout()
{
    const QString text = QString::fromUtf8(m_process->readAllStandardOutput());
    m_buffer += text;
    m_log->append(text.trimmed());
}

void XxxDialog::onFinished(int exitCode, QProcess::ExitStatus status)
{
    m_buffer += QString::fromUtf8(m_process->readAllStandardOutput());

    m_runBtn->setEnabled(true);
    m_runBtn->setText(QStringLiteral("运行"));

    if (status != QProcess::NormalExit || exitCode != 0) {
        m_log->append(QStringLiteral("\n执行失败，退出码 %1。").arg(exitCode));
    }
    showResult(m_buffer);

    if (m_process) { m_process->deleteLater(); m_process = nullptr; }
}

// 从输出里倒着找第一行能解析的 JSON
void XxxDialog::showResult(const QString& output)
{
    const QStringList lines = output.split('\n', Qt::SkipEmptyParts);
    for (int i = lines.size() - 1; i >= 0; --i) {
        const QString line = lines.at(i).trimmed();
        if (!line.startsWith('{')) continue;

        QJsonParseError err;
        const QJsonDocument doc = QJsonDocument::fromJson(line.toUtf8(), &err);
        if (err.error != QJsonParseError::NoError || !doc.isObject()) continue;

        const QJsonObject obj = doc.object();
        if (!obj.value("ok").toBool()) {
            m_log->append(QStringLiteral("\n算法返回错误: ") + obj.value("error").toString());
            return;
        }
        m_log->append(QStringLiteral("\n---- 结果 ----"));
        m_log->append(QStringLiteral("行数: %1").arg(obj.value("rows").toInt()));
        m_log->append(QStringLiteral("结果文件: ") + obj.value("csv").toString());
        return;
    }
}
```

### 3.3 关键点

- **路径一律相对 `applicationDirPath()`**，也就是 `bin_qt_release_64\`。写成绝对路径换机器就废。
- **`QProcess::MergedChannels`**：Python 的 `print` 和报错都进同一个管道，日志区不会缺行。
- **`PYTHONUTF8=1`**：中文输出必须，否则 Windows 默认 GBK 会乱码。
- **工作目录设成脚本所在目录**：脚本里如果用相对路径读同目录的模型/配置才不会挂。
- 如果算法需要**进程内**调用 Python（共享内存、传大数据），可以改用工程里已有的 `py_wrapper.h/.cpp`；本文的 `QProcess` 方式更简单、更安全（脚本崩了不会带崩主程序）。

---

## 4. 第三步：在 Ribbon 上加按钮

涉及 `modelTool1RibbonCategory.h` / `.cpp`，共 4 处改动。

### 4.1 头文件：加槽声明

```cpp
public slots:
    void newBt1Slot();
    ...
    void newBt9Slot();
    void newBt10Slot();      // ← 新增
```

### 4.2 cpp 顶部：包含新对话框头文件

```cpp
#include "LasStatDialog.h"
#include "XxxDialog.h"       // ← 新增
```

### 4.3 cpp 的 `PrivateData`：加按钮指针

```cpp
QAction* mbt9  = nullptr;    //9
QAction* mbt10 = nullptr;    //10   ← 新增
```

### 4.4 `createTools()`：创建动作 + 连接 + 加到面板

```cpp
d_ptr->mbt10 = new QAction(QIcon(btLogo10), QStringLiteral("我的新算法"), this);
d_ptr->mbt10->setToolTip(QStringLiteral("我的新算法"));
connect(d_ptr->mbt10, SIGNAL(triggered()), this, SLOT(newBt10Slot()));
...
d_ptr->m_pPannel_NewObject->addLargeAction(d_ptr->mbt10);
```

### 4.5 实现槽

```cpp
void CModelTool1RibbonCategory::newBt10Slot()
{
    XxxDialog dlg(this);
    dlg.exec();          // 模态
}
```

### 4.6 图标

- 复用现成图标：直接写 `QIcon(btLogo9)`，跳过本节。
- 新图标：把 SVG 放到 `bin_qt_release_64\resource\Subtopic\`，文件名随意（例如 `10我的新算法.svg`），然后在 `subtopicByPython_i.h` 里加一行：

```cpp
#define btLogo10 defExePath +"/resource/Subtopic/10我的新算法" + SVG
```

> `defExePath` 和 `SVG` 已在该头文件里定义，图标是**运行时**从 exe 目录读的，不需要编译进资源。

---

## 5. 第四步：把文件加进工程

### 5.1 推荐做法（VS 里操作）

1. 把 `XxxDialog.h` / `XxxDialog.cpp` 放到 `QSubtopicByPython\` 目录下
2. 解决方案资源管理器 → 右键项目 → **添加 → 现有项** → 选中两个文件
3. **关键一步**：选中 `XxxDialog.h` → 右键 **属性** → **项类型** 改成 **Qt Moc**
   （不改的话 VS 不会为它生成 `moc_XxxDialog.cpp`，链接时报 `LNK2019 无法解析的外部符号 staticMetaObject` / vtable）

### 5.2 手工改 vcxproj（等价）

在 `QSubtopicByPython.vcxproj` 的 `<ItemGroup>` 里加：

```xml
<ItemGroup>
  <ClCompile Include="XxxDialog.cpp" />
  <QtMoc Include="XxxDialog.h" />
</ItemGroup>
```

如果用了 `.ui` 文件，再加：

```xml
<QtUic Include="XxxDialog.ui" />
```

> `QSubtopicByPython.pri` / `.pro` 是 Qt VS Tools 生成的副产物，**不要手改**，改 vcxproj 即可。

---

## 6. 第五步：编译

### 6.1 先关掉正在运行的软件

`QApp.exe` 运行时 `QSubtopicByPython.dll` 被占用，链接会报：

```
LNK1104: 无法打开文件 "...\plugins_user\QSubtopicByPython.dll"
```

### 6.2 VS 里编译

1. 打开 `DiBaiYanJiuYuanWork\GSSApp.sln`
2. 顶部配置选 **Release** + **x64**
3. 右键 `QSubtopicByPython` → **生成**（不用编整个解决方案）

### 6.3 命令行编译（等价）

```powershell
& 'C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe' `
  'D:\迪拜研究院\DiBaiYanJiuYuanCodeAndRX64\DiBaiYanJiuYuanWork\SubtopicByPython\QSubtopicByPython\QSubtopicByPython.vcxproj' `
  /t:Build /p:Configuration=Release /p:Platform=x64 `
  "/p:QtMsBuild=C:\Users\<用户名>\AppData\Local\Microsoft\VisualStudio\17.0_<id>\Extensions\<扩展目录>\QtMSBuild" `
  /m /v:m
```

`QtMsBuild` 的实际路径 = `%LOCALAPPDATA%\Microsoft\VisualStudio\17.0_*\Extensions\*\QtMSBuild`。

### 6.4 成功的标志

输出末尾出现：

```
  moc XxxDialog.h
  XxxDialog.cpp
  QSubtopicByPython.vcxproj -> D:\...\bin_qt_release_64\plugins_user\QSubtopicByPython.dll
```

并且 `bin_qt_release_64\plugins_user\QSubtopicByPython.dll` 的**时间戳是最新的**。这一步最容易出问题——只要时间戳没变，后面怎么点按钮都不会有效果。

---

## 7. 第六步：运行验证

1. 启动 `D:\迪拜研究院\DiBaiYanJiuYuanCodeAndRX64\bin_qt_release_64\QApp.exe`
2. Ribbon → **SubtopicByPython** 选项卡 → **基础工具** 面板 → 找到新按钮（排在最后）
3. 点开 → 选输入文件 → 选输出目录 → **运行**

以 LasStat 为例，正常日志：

```
[las_stat] 读取 D:\...\TEST-001.las
[las_stat] 共 301 行 / 5 条曲线
[las_stat] 统计表已写入 D:\...\out\TEST-001_curve_stat.csv

---- 统计结果 ----
数据行数: 301    曲线数: 5
深度段: 1000 ~ 1300 m
DEPT  n=301  min=1000  max=1300  mean=1150  std=86.8907
GR    n=301  min=23.44 max=115.97 mean=73.371 std=25.249
...
统计表: D:\...\TEST-001_curve_stat.csv
```

测试数据可用 `testdata\TEST-001.las`（301 行、5 条曲线，含 `-999.25` 空值段）。

---

## 8. 常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| 按钮没出现 | DLL 没更新，或加载失败 | 看 dll 时间戳；关掉 QApp 重编 |
| `LNK1104 无法打开 ...QSubtopicByPython.dll` | 软件还开着 | 关闭 `QApp.exe` 后重编 |
| `LNK2019 无法解析 staticMetaObject / vtable` | `.h` 没设为 Qt Moc | 属性 → 项类型 → Qt Moc，或手加 `<QtMoc>` |
| `找不到 Qt 版本 QT_6.4.3_msvc2019_X64` | 没注册或名字不一致 | Qt VS Tools → Qt Versions 里 Add，名字必须完全一致 |
| 点按钮无反应 | `connect` 没写 / 槽没声明在 `public slots` / 忘了 `Q_OBJECT` | 检查三处 |
| 日志中文乱码 | 没设 `PYTHONUTF8` | `env.insert("PYTHONUTF8", "1")` |
| 日志显示"找不到脚本" | 脚本没拷到 `python2_evn\Subtopic\<模块>\` | 脚本不参与编译，要单独放 |
| 改了 `.py` 但结果没变 | 脚本是运行时读取的，但**要重启软件** | 关掉 QApp 重新打开（无需重编） |
| Debug 编译报一堆找不到头文件 | Debug 配置不完整 | 只用 Release\|x64 |

---

## 9. 改动清单（照着核对）

新增一个模块，最小改动集合：

```
[ ] bin_qt_release_64\python2_evn\Subtopic\<模块>\xxx.py        新建
[ ] QSubtopicByPython\XxxDialog.h                                新建
[ ] QSubtopicByPython\XxxDialog.cpp                              新建
[ ] QSubtopicByPython\modelTool1RibbonCategory.h                 改：槽声明 + PrivateData 指针
[ ] QSubtopicByPython\modelTool1RibbonCategory.cpp               改：include / createTools / 槽实现
[ ] QSubtopicByPython\QSubtopicByPython.vcxproj                  改：ClCompile + QtMoc
[ ] bin_qt_release_64\resource\Subtopic\<图标>.svg                可选，新图标
[ ] QSubtopicByPython\subtopicByPython_i.h                       可选，btLogoN 宏
```

编译产物只有一个：`bin_qt_release_64\plugins_user\QSubtopicByPython.dll`。

---

## 附：LasStat 参考实现

| 文件 | 说明 |
|---|---|
| `python2_evn\Subtopic\LasStat\las_stat.py` | 脚本：解析 LAS/CSV，算 min/max/mean/std，写 CSV，输出 JSON |
| `QSubtopicByPython\LasStatDialog.h/.cpp` | 对话框：输入文件 + 输出目录 + 日志 + 运行 |
| `QSubtopicByPython\modelTool1RibbonCategory.cpp:96` | 第 9 个按钮的创建 |
| `QSubtopicByPython\modelTool1RibbonCategory.cpp:188` | `newBt9Slot()` 打开对话框 |
| `QSubtopicByPython\subtopicByPython_i.h:31` | `btLogo9` 图标宏 |
| `QSubtopicByPython\QSubtopicByPython.vcxproj:116,118` | `ClCompile` / `QtMoc` 登记 |
