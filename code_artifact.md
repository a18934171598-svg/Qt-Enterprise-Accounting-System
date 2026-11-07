# 📊 小微企业财务系统 (Enterprise Accounting System)

![C++](https://img.shields.io/badge/C++-17-blue.svg)
![Qt](https://img.shields.io/badge/Qt-6.0+-green.svg)
![SQLite](https://img.shields.io/badge/Database-SQLite3-lightgrey.svg)
![Platform](https://img.shields.io/badge/Platform-Windows-blue)

> **⚠️ 声明：**
> 本项目为商业化落地的独立单机版财务软件（已进行版权保护与软著申请），此处仅作为 **Showcase（展示库）** 公开核心架构设计、业务逻辑处理方案以及部分非涉密代码片段。完整闭源系统暂不提供直接下载。

---

## 📖 项目简介
本项目是专为小微企业设计的一款桌面端本地财务记账系统。严格遵循《2013小企业会计准则》，实现了从**期初建账、日常凭证录入、附件管理**到**月末结账、自动生成三大报表（资产负债表、利润表、现金流量表）**的完整财务闭环。

针对传统单机软件易丢失数据、环境配置复杂的痛点，系统采用了**本地轻量级数据库 + 相对路径附件隔离**的持久化架构，并内置了一键打包备份与防篡改机制，实现了真正的“开箱即用”与“数据随身走”。

---

## 🚀 核心架构与技术亮点

### 1. 极致的财务数据防丢精度 (Financial Precision)
*   **分币级存储架构：** 彻底摒弃 `float/double` 的浮点数存储，数据库与底层 C++ 业务逻辑全面采用 `int64_t` 按“分”为单位进行计算与存储，从根本上杜绝了金额累计时的 `0.0000001` 精度丢失问题。
*   **UI 渲染妆匣机制：** 采用 Qt 的 `QStyledItemDelegate`（Model/View 架构），在底层数据保持纯净 `int64_t` 的同时，UI 渲染层动态加载正则过滤与千位分隔符（如 `1,234,567.89`），实现数据存储与视图渲染的绝对解耦。

### 2. 高性能跨平台附件渲染引擎 (Gallery Engine)
*   放弃了传统的 `QLabel` 图片显示，深度封装了基于 `QGraphicsScene` 和 `QGraphicsView` 的二维图形渲染引擎。
*   原生支持**鼠标滚轮无极缩放、抓手工具平移**，并开启抗锯齿（Antialiasing）与平滑像素变换（SmoothPixmapTransform），即便加载超高分辨率的发票原图或 PDF，也能保证极速响应与零卡顿。

### 3. 工程化的数据隔离与一键灾备 (Data Persistence & Backup)
*   **双轨路径策略：** 针对票据附件，自研了“物理绝对路径写入 + 逻辑相对路径存库”的机制。确保用户将软件打包发送到其他电脑时，数据库与附件的映射关系绝不失效。
*   **无阻塞环境隔离打包：** 利用 `QProcess` 异步调用底层 OS 原生 `tar` 命令。通过多线程与事件循环防止海量附件压缩时导致的 UI 假死，并设计了 300 秒超时熔断机制保障进程安全。

### 4. 商业级安全与权限控制 (Security & DRM)
*   引入基于 Windows 底层密码学 GUID 的机器码指纹提取机制。
*   结合内部加盐（Salt）与国安局级别 SHA-256 算法生成设备唯一激活密钥，实现“一机一码”防盗版拦截。

---

## 💻 核心模块展示

### 1. 智能凭证录入与现金流追踪

<img width="606" height="910" alt="image" src="https://github.com/user-attachments/assets/8c46f312-2f66-4c72-86ac-661acaa311ac" />

*   **智能拦截：** 多级验证必填项、借贷必须平衡方可入库。
*   **现金流自动感知：** 核心逻辑会自动判断科目是否触碰“资金池”（1001现金/1002银行存款）。如果是单边资金变动，强制阻塞弹窗，要求出纳指定现金流向（经营/投资/筹资），确保生成的现金流量表绝对准确。

### 2. 试算平衡与期初建账 (Trial Balance)

<img width="1585" height="997" alt="image" src="https://github.com/user-attachments/assets/fa036808-7e50-4745-935d-71f29989068e" />

>
> *   支持自下而上的树状科目锁定，包含下级科目的父节点自动变灰且禁止录入。
> *   严格执行“借贷必等”的铁律，差一分钱系统都会拒绝建账锁定。

### 3. 月末结账与沙盒管理
<img width="1519" height="667" alt="image" src="https://github.com/user-attachments/assets/6d3a6a77-5ede-4b04-85ea-5bcec4239549" />

*   利用 SQLite 单表双锚点时间机器记录系统状态。
*   支持严格的逐月封账与反结账操作，封账后的月份进入只读沙盒模式，所有凭证不可修改、不可删除，保障审计留痕。

---

## 🛠 代码片段 (Code Snippets)

由于商业机密，此处仅展示部分非核心业务逻辑但能体现工程化思维的基础代码：

### 异步灾备打包 (防止 UI 阻塞)
```cpp
// 核心黑科技:静默调用 Windows 原生 tar 命令进行环境隔离打包
QProcess process;
process.setWorkingDirectory(appDir); // 确保解压后无冗余外层目录
QStringList args;
args << "-a" << "-c" << "-f" << savePath << dbFileName << attachDirName;

this->setEnabled(false); // 禁用主窗口，防止打包期间数据写入冲突
QApplication::setOverrideCursor(Qt::WaitCursor); // 情绪安抚
process.start("tar", args); 

// 容错: 300秒超时熔断。防止上万张发票导致压缩过久，软件假死
if (!process.waitForFinished(300000)) {
    this->setEnabled(true);
    process.kill(); // 强行杀掉底层卡死的压缩进程
    QMessageBox::warning(this, "超时", "打包耗时过长已被中断。");
    return;
}
```

### 彻底消灭浮点误差的 Model/View 渲染代理
```cpp
class MoneyDelegate : public QStyledItemDelegate {
public:
    // 用户双击输入时，唤醒带正则拦截的编辑器
    QWidget* createEditor(QWidget* parent, const QStyleOptionViewItem& option, const QModelIndex& index) const override {
        QLineEdit* editor = new QLineEdit(parent);
        // 完美限制整数与两位小数
        QRegularExpression regx("^(0|[1-9][0-9]{0,15})(\\.[0-9]{1,2})?$");
        editor->setValidator(new QRegularExpressionValidator(regx, editor));
        return editor;
    }

    // 静态展示时，为底层的分币数据穿上千位分隔符外套
    QString displayText(const QVariant& value, const QLocale& locale) const override {
        QString text = value.toString().trimmed();
        if (text.isEmpty()) return ""; 
        double val = text.toDouble();
        return QLocale(QLocale::English).toString(val, 'f', 2);
    }
};
```
