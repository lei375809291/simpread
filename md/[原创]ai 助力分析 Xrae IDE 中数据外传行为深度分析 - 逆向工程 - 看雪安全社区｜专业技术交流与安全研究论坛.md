> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293020.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

**分析版本**：构建版本 `2.3.85576` / 内核版本 `1.107.1` (VSCode Fork) / 应用版本 `3.3.102`  
**分析环境**：Windows 11 x64

**关键字已用 X 脱敏，有什么问题找 ZCODE 用这个辅助逆的。**

1.  [执行摘要与调查结论](#%E4%B8%80%E6%89%A7%E8%A1%8C%E6%91%98%E8%A6%81%E4%B8%8E%E8%B0%83%E6%9F%A5%E7%BB%93%E8%AE%BA)
2.  [客户端架构与核心二进制通信矩阵](#%E4%BA%8C%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%9E%B6%E6%9E%84%E4%B8%8E%E6%A0%B8%E5%BF%83%E4%BA%8C%E8%BF%9B%E5%88%B6%E9%80%9A%E4%BF%A1%E7%9F%A9%E9%98%B5)
3.  [核心专题：云控能否绕过隐私权限强制上传代码？](#%E4%B8%89%E6%A0%B8%E5%BF%83%E4%B8%93%E9%A2%98%E4%BA%91%E6%8E%A7%E8%83%BD%E5%90%A6%E7%BB%95%E8%BF%87%E9%9A%90%E7%A7%81%E6%9D%83%E9%99%90%E5%BC%BA%E5%88%B6%E4%B8%8A%E4%BC%A0%E4%BB%A3%E7%A0%81)
4.  [CKG 远程向量化与源码打包上传逆向分析 (libckg.dll)](#%E5%9B%9Bckg-%E8%BF%9C%E7%A8%8B%E5%90%91%E9%87%8F%E5%8C%96%E4%B8%8E%E6%BA%90%E7%A0%81%E6%89%93%E5%8C%85%E4%B8%8A%E4%BC%A0%E9%80%86%E5%90%91%E5%88%86%E6%9E%90-libckgdll)
5.  [AI 补全上下文截获与 relevantFiles.zip 上传逆向分析 (cueMain.js)](#%E4%BA%94ai-%E8%A1%A5%E5%85%A8%E4%B8%8A%E4%B8%8B%E6%96%87%E6%88%AA%E8%8E%B7%E4%B8%8E-relevantfileszip-%E4%B8%8A%E4%BC%A0%E9%80%86%E5%90%91%E5%88%86%E6%9E%90-cuemainjs)
6.  [底层 Native 遥测与云控检索系统逆向分析](#%E5%85%AD%E5%BA%95%E5%B1%82-native-%E9%81%A5%E6%B5%8B%E4%B8%8E%E4%BA%91%E6%8E%A7%E6%A3%80%E7%B4%A2%E7%B3%BB%E7%BB%9F%E9%80%86%E5%90%91%E5%88%86%E6%9E%90)
7.  [“关闭遥测后仍产生网络流量” 的机理溯源](#%E4%B8%83%E5%85%B3%E9%97%AD%E9%81%A5%E6%B5%8B%E5%90%8E%E4%BB%8D%E4%BA%A7%E7%94%9F%E7%BD%91%E7%BB%9C%E6%B5%81%E9%87%8F%E7%9A%84%E6%9C%BA%E7%90%86%E6%BA%AF%E6%BA%90)
8.  [隐私暴露风险矩阵与全链路审计](#%E5%85%AB%E9%9A%90%E7%A7%81%E6%9A%B4%E9%9C%B2%E9%A3%8E%E9%99%A9%E7%9F%A9%E9%98%B5%E4%B8%8E%E5%85%A8%E9%93%BE%E8%B7%AF%E5%AE%A1%E8%AE%A1)
9.  [今天敢偷源码，明天就敢挖矿](#%E4%B9%9D%E4%BB%8A%E5%A4%A9%E6%95%A2%E5%81%B7%E6%BA%90%E7%A0%81%EF%BC%8C%E6%98%8E%E5%A4%A9%E5%B0%B1%E6%95%A2%E6%8C%96%E7%9F%BF)

针对近期关于某节跳动旗下 AI IDE 产品 Xrae（国内版 Xrae CN）在用户关闭遥测及隐私权限后依然持续产生后台网络上传行为的问题，本报告基于静态反汇编、反编译、Go/Rust 符号表恢复、字符串交叉引用及配置文件逆向，深入追踪 Xrae 客户端的网络行为与数据流向。

核心结论如下：

1.  **云控拥有绝对覆盖权（可以强制开启代码上传）**：即使用户在本地关闭隐私 / 遥测权限，且启动脚本配置了 `-local_embedding`，Xrae 的代码架构中也已完整实现了**云端 FeatureGate / ABConfig 强行覆盖本地配置并开启远程代码上传**的完整闭环（`LocalRemoteEmbeddingSelector` 动态选择分支）。
2.  **远程 Embedding 机制客观存在**：在 CKG（代码知识图谱）核心动态库 `libckg.dll` 中，完整保留并实现了**遍历本地工程、读取源码明文、批量分片并向云端上传代码文件**以构建远程代码知识库的完整逻辑（`CollectFilesAndRemoteEmbeddingStep`）。
3.  **补全引擎隐蔽外传剪贴板与终端信息**：补全引擎 `cueMain.js` 在构建每次补全请求时，会无差别读取**操作系统剪贴板（Clipboard）明文**及**集成终端的命令与编辑历史（`terminal_edit`）**作为上下文回传，该链路**没有任何针对隐私模式的校验判断**。
4.  **特定事件直接打包上传源码压缩包（`relevantFiles.zip`）**：在事件上报系统 `ReportEventApi` 中，当云端云控开启特定开关时，客户端会将当前关联的代码文件打包为 `relevantFiles.zip`，以二进制方式直接 `POST` 上传至服务端。
5.  **底层 Native 组件与前端遥测开关脱钩**：用户在界面关闭 `telemetry.telemetryLevel` 仅能静音 VSCode 原生扩展层事件。底层的 TTNet（`sscronet.dll`）、APM 性能监控（Slardar）、设备指纹（MetaSec）与云控日志拉取器（Logifier）均绕过该开关独立运行。

Xrae IDE 虽基于 VSCode 开源框架二次开发，但其核心网络与 AI 逻辑已被完全重构，替换为 X 节跳动自研的 Native C++/Rust/Go 组件：

```
flowchart TD
    Client["Trae IDE 客户端 (宿主进程)"]

    Client --> CKG["代码知识图谱 (CKG 引擎) - libckg.dll"]
    Client --> AIChat["AI 补全与智能调度 - cueMain.js / extension.js"]
    Client --> NativeNet["底层 Native 监控网络 - sscronet / metasecml / logifier"]

    subgraph CKG_Group["CKG 知识图谱引擎"]
        CKG --> CKG_Local["本地向量化 - LocalChunkStep (sqlite_vec)"]
        CKG --> CKG_Remote["远程源码上传切片 - RemoteEmbedStep (云端建库)"]
    end

    subgraph AI_Group["AI 补全与调度模块"]
        AIChat --> AI_Ctx["敏感上下文提取 - 剪贴板 / 终端记录 / 文件树"]
        AIChat --> AI_Zip["源码包直接上传 - ReportEventApi (relevantFiles.zip)"]
    end

    subgraph Net_Group["底层网络与监控系统"]
        NativeNet --> Net_Probe["HTTPDNS / TNC 探针 (dig.bdurl.net / zijieapi)"]
        NativeNet --> Net_Sec["设备指纹与云控日志检索 (log.snssdk.com / logifier)"]
    end

    style Client fill:#1e293b,stroke:#3b82f6,stroke-width:2px,color:#fff
    style CKG fill:#0f172a,stroke:#6366f1,stroke-width:1.5px,color:#e2e8f0
    style AIChat fill:#0f172a,stroke:#ec4899,stroke-width:1.5px,color:#e2e8f0
    style NativeNet fill:#0f172a,stroke:#f59e0b,stroke-width:1.5px,color:#e2e8f0
    style CKG_Local fill:#1e293b,stroke:#10b981,stroke-width:1px,color:#cbd5e1
    style CKG_Remote fill:#450a0a,stroke:#ef4444,stroke-width:1.5px,color:#fca5a5
    style AI_Ctx fill:#450a0a,stroke:#ef4444,stroke-width:1.5px,color:#fca5a5
    style AI_Zip fill:#450a0a,stroke:#ef4444,stroke-width:1.5px,color:#fca5a5
    style Net_Probe fill:#1e293b,stroke:#eab308,stroke-width:1px,color:#cbd5e1
    style Net_Sec fill:#450a0a,stroke:#ef4444,stroke-width:1.5px,color:#fca5a5
```

<table><thead><tr><th>业务通道</th><th>目标端点 / 域名</th><th>API 路径</th><th>传输内容与目的</th></tr></thead><tbody><tr><td><strong>代码文件外传</strong></td><td><code>api.*rae.com.cn</code></td><td><code>/api/ide/v1/report/clients</code></td><td>发生特定事件时，以 <code>multipart/form-data</code> 上传 <code>relevantFiles.zip</code></td></tr><tr><td><strong>远程知识库建库</strong></td><td><code>api.*rae.com.cn</code></td><td><code>/knowledgebase/upload</code> (CKG)</td><td>远程 Embedding 模式下，分批上传项目源代码文件明文</td></tr><tr><td><strong>云控日志拉取</strong></td><td><code>api.*rae.com.cn</code></td><td><code>/logifier/retrieval/tasks</code><br><code>/logifier/files//parts/</code></td><td>云端下发检索规则，客户端打包本地日志（<code>.tar.gz</code> / <code>.lgpk</code>）分片上传</td></tr><tr><td><strong>设备注册与指纹</strong></td><td><code>log.*****.com</code></td><td><code>/service/2/desktop/device_register/</code></td><td>上传网卡 MAC 哈希、驱动列表、屏幕参数生成的固化设备 ID</td></tr><tr><td><strong>APM 性能监控</strong></td><td><code>pc-mon.*****api.com</code><br><code>mon.*****api.com</code></td><td><code>/monitor_pc/collect/api/pc_crash</code><br><code>/monitor_pc/collect/api/pc_jank</code></td><td>进程崩溃 MiniDump（含内存镜像）、界面卡顿、基础性能指标</td></tr><tr><td><strong>网络调度与探针</strong></td><td><code>tnc3-bjlgy.*****api.com</code></td><td>TNC 调度参数</td><td>TTNet 底层周期性探测与测速数据包</td></tr></tbody></table>

针对用户最为关注的问题：“**是不是用户哪怕关闭隐私权限，Xrae 也可以通过云控开启，然后上传用户代码？**”

基于严谨的符号推导与代码逻辑交叉引用，结论非常明确：**是的，完全可以，且在代码架构上已经完整实现了该闭环。**

在 `resources/app/modules/ckg/start.bat` 第 82 行，客户端虽然传入了 `-local_embedding` 与 `-embedding_storage_type=sqlite_vec`；但在底层 `libckg.dll` 中，通过符号表提取到了核心决策模块：

*   **符号**：`ide/ckg/codekg/components/selector.LocalRemoteEmbeddingSelector`（`RawOffset=0x01002F8E`, `RVA=0x01004F8E`）
*   **关联响应**：`*knowledgebase.GetABConfigResponse`、`*knowledgebase.GetFeaturesConfigResponse`

**决策控制流推导**：

```
// 逆向还原: ide/ckg/codekg/components/selector/selector.go
func (s *LocalRemoteEmbeddingSelector) SelectPipelineStep(ctx context.Context, proj *Project) pipeline.Step {
    // 1. 获取云端下发的 AB 实验与特性配置
    abConfig := s.client.GetABConfig(ctx)
    features := s.client.GetFeaturesConfig(ctx)

    // 2. 关键判断：云控策略拥有最高优先级
    // 即使启动时命令行传入了 -local_embedding，云端若将 enable_local_embedding 置为 false
    if !abConfig.EnableLocalEmbedding || !features.AllowLocalIndex {
        log.Info("Cloud control disabled local embedding, switching to remote upload pipeline")
        // 强行实例化远程上传流水线！
        return index.NewCollectFilesAndRemoteEmbeddingStep(s.client, proj)
    }

    // 3. 项目规模超限检查
    if proj.TotalFileSize > features.FileSizeThreshold {
        log.Warn("Project size exceeds threshold, fallback to remote upload")
        return index.NewCollectFilesAndRemoteEmbeddingStep(s.client, proj)
    }

    // 4. 仅当云端允许且未超限时，才走本地 sqlite_vec
    return index.NewLocalChunkAndEmbeddingStep(proj)
}
```

**分析**：本地启动参数仅是 “初始默认值”，代码层面的最高仲裁权由服务端的 `GetABConfig` 与 `GetFeaturesConfig` 把控。只要云端云控调整配置，流水线会直接流向 `CollectFilesAndRemoteEmbeddingStep`，执行本地代码全量读取与上传。

在 `cueMain.js` 约 3111815 偏移处组装补全请求体时：

*   剪贴板读取（`ClipboardManager.getClipboardText()`）、终端记录（`terminal_edit`）以及工程文件树（`relevantFileManager.getContext()`）被组装为 `requestBody`；
*   **该代码块内没有任何一处调用 `isTelemetryEnabled()` 或 `isPrivacyMode()`**；
*   架构设计将这些用户敏感数据定义为 “模型补全所必需的上下文（Inference Context）”，从而在逻辑分支上天然绕过了所有隐私 / 遥测开关。

在 `extension.js`（`RawOffset=0x00027524`）中定义了由云端 Libra 平台下发的动态布尔值开关：

*   `FeatureName.ENABLE_REPORT_CLIENT_DATA = "enable_report_client_data"`
*   `FeatureName.ENABLE_CUE_CONTEXT_REPORT = "enable_cue_context_report"`

当云端将这两个云控开关下发为 `true` 时，`cueMain.js` 中的 `ReportEventApi` 会在捕获到特定追踪事件时，自动将上下文关联的源代码文件压缩为 **`relevantFiles.zip`** 并直接发起 `POST` 上传。

`resources/app/modules/ckg/binary/libckg.dll` 是基于 Go 1.25 编译的 PE 动态链接库。

通过 PE 头解析获取到 `libckg.dll` 的节区信息与关键符号物理地址：

```
Sections for libckg.dll:
Section .text  : VAddr=0x00001000 RawOffset=0x00000400 RawSize=0x00EE0800
Section .data  : VAddr=0x00EE2000 RawOffset=0x00EE0C00 RawSize=0x000BF400
Section .rdata : VAddr=0x00FA2000 RawOffset=0x00FA0000 RawSize=0x01527200
Section .pdata : VAddr=0x024CA000 RawOffset=0x024C7200 RawSize=0x00057400
```

在 `.rdata` 节区中，精确提取到以下关键符号与类型元数据地址：

*   `CollectFilesAndRemoteEmbeddingStep` : `RawOffset=0x010092B2`, `RVA=0x0100B2B2`
*   `BatchUploadKnowledgebaseFilesTask` : `RawOffset=0x010064DD`, `RVA=0x010084DD`
*   `UploadKnowledgebaseFilesRequest` : `RawOffset=0x01012FA1`, `RVA=0x01014FA1`
*   `LocalRemoteEmbeddingSelector` : `RawOffset=0x01002F8E`, `RVA=0x01004F8E`
*   `readFileContent` : `RawOffset=0x00FC1B86`, `RVA=0x00FC3B86`
*   `LocalChunkAndEmbeddingStep` : `RawOffset=0x00FF8B04`, `RVA=0x00FFAB04`
*   `UploadKnowledgebaseFiles` : `RawOffset=0x00FE20DC`, `RVA=0x00FE40DC`

在符号表中提取到编译源文件路径（开发机标识 `C:/673**/ckg/`）：

*   `C:/673**/ckg/codekg/components/pipeline_steps/index/collect_files.go`
*   `C:/673**/ckg/codekg/components/pipeline_steps/index/collect_files_and_remote_embedding.go`
*   `C:/673**/ckg/codekg/components/pipeline_steps/index/remote_embedding.go`
*   `C:/673**/ckg/codekg/components/pipeline_steps/index/local_chunk_and_embedding.go`
*   `C:/673**/ckg/codekg/components/knowledgebase/client.go`
*   `C:/673**/ckg/codekg/components/periodic/upload_file_limit.go`

```
package knowledgebase

// 上传单文件结构体
// 符号 RVA: 0x010084DD
type UploadKnowledgebaseFile struct {
    FileID      string `json:"file_id"`
    FilePath    string `json:"file_path"`
    Content     string `json:"content"`      // 用户源代码全文明文
    Language    string `json:"language"`
    Size        int64  `json:"size"`
    MD5         string `json:"md5"`
}

// 批量上传请求
// 符号 RVA: 0x01014FA1
type UploadKnowledgebaseFilesRequest struct {
    ProjectID           string                    `json:"project_id"`
    KnowledgebaseURI    string                    `json:"knowledgebase_uri"`
    Files               []UploadKnowledgebaseFile `json:"files"`
    IndexStrategy       int                       `json:"index_strategy"`
    UploadType          string                    `json:"upload_type"`
}

// 任务执行器
// 符号 RVA: 0x00FC3B86
func (t *BatchUploadKnowledgebaseFilesTask) readFileContent(filePath string) ([]byte, error) {
    // 汇编调用链: os.ReadFile -> io.ReadAll -> 填入 UploadKnowledgebaseFile.Content
}
```

**汇编执行特征**：  
在 `BatchUploadKnowledgebaseFilesTask.do` 的执行循环中：

```
; 伪汇编: 遍历项目文件并读取内容
lea     rdx, [rbp+filePath]      ; 加载文件路径
call    os.ReadFile              ; 读取源码明文到内存切片
mov     [rsp+fileContent], rax   ; 存入待上传结构体
call    UploadKnowledgebaseFiles ; 发起 HTTP POST 请求
```

在 `resources/app/extensions/ai-completion/resource/aiserver/cueMain.js` 中，实现了前端补全逻辑、上下文提取与事件上报。

在 `cueMain.js` 约 3111815 偏移处，逆向反混淆后的补全上下文构建逻辑：

```
// 补全请求上下文组装逻辑 (cueMain.js Offset: 3111815)
async assembleCompletionContext(document, position, options) {
    let clipboardText = await this.clipboardManager.getClipboardText(); // 读取剪贴板
    let relevantFilesContext = await this.relevantFileManager.getContext(document); // 遍历工程目录树

    let requestBody = {
        user_behavior_code:  contextData?.behaviorContext,  // 用户近期击键与光标行为
        neighbor_snippet:    contextData?.neighborSnippets, // 邻近标签页代码片段
        embedding_snippet:   "",                            // 向量召回片段

        // 关键点 1: 剪贴板内容被序列化上报 (极高敏感度)
        clipboard: clipboardText ? JSON.stringify({ content: clipboardText }) : undefined,

        file_path_edit:      this.getRenameContext(),       // 文件重命名历史
        chat_summary:        this.chatSummary,              // AI 聊天摘要

        // 关键点 2: 终端执行与编辑记录 (包含命令行参数、执行输出)
        terminal_edit:       options.terminalInfo,

        // 关键点 3: 关联工程文件列表
        relevant_files:      relevantFilesContext
    };

    if (options.bizContext) {
        requestBody.biz_context = {
            repo_urls: options.bizContext.repo_urls         // 用户的 Git Remote URL
        };
    }

    return requestBody;
}
```

在 `cueMain.js` 约 3100002 偏移处，`relevantFileManager` 遍历用户工作区的具体实现：

```
// cueMain.js Offset: 3100002
async getContext(e) {
    let r = Date.now();
    try {
        let n = [],
            s = _.DocumentUtils.getFilePath(e.uri),
            o = path.dirname(s),
            a = new Set;
        await this.getDirFiles(s, a);
        for (let l of this.editFileList.keys()) await this.getDirFiles(l, a);
        for (let l of this.openFileList.keys()) await this.getDirFiles(l, a);
        let c = _.DocumentUtils.getWorkspacePath(e, U.workspaceFolders);
        for (let l of a) {
            l.startsWith(c) && n.push({
                path: l.substring(c.length + 1),
                edited: this.editFileList.has(l),
                neighbor: path.dirname(l) === o
            });
        }
        return JSON.stringify({
            folder_files: n,
            open_files: [...this.openFileList.keys()]
        });
    } catch {}
}
```

`cueMain.js` 中内置了系统剪贴板监听组件：

```
class ClipboardManager {
    constructor() {
        this.clipboardCache = null;
        this.CACHE_TTL_MS = 500;
    }
    async getClipboardText() {
        let now = Date.now();
        if (this.clipboardCache && (now - this.clipboardCache.timestamp < this.CACHE_TTL_MS)) {
            return this.clipboardCache.content;
        }
        try {
            // 通过 IPC 请求获取宿主操作系统剪贴板内容
            let text = (await this.connection?.sendRequest("GetClipboardTextRequest"))?.text ?? "";
            this.clipboardCache = { content: text, timestamp: now };
            return text;
        } catch (err) {
            return "";
        }
    }
}
```

**隐私危害**：开发者在日常编码中经常将数据库密码、API Token、私钥或敏感配置复制到剪贴板。`ClipboardManager` 会在用户进行常规代码编辑触发补全时，自动抓取剪贴板内容并提交至服务端。

在 `cueMain.js` 约 1135122 偏移处及 `extension.js` 约 37621 偏移处，定义了客户端事件上报的底层传输接口：

```
// 客户端事件上报与文件外传接口 (cueMain.js Offset: 1135122)
class ReportEventApi extends CommonApi {
    constructor() {
        super();
        this.path = "api/ide/v1/report/clients";
        this.method = HttpMethod.POST;
        this.headers = {
            "Content-Type": "multipart/form-data"
        };
        this.checkJWTToken = true;
    }

    transformRequest(params) {
        let { env_metadata, file, event } = params;
        let formData = new FormData();
        let payload = {
            env_metadata: env_metadata,
            events: [event]
        };
        formData.append("body", JSON.stringify(payload));

        // 关键点: 将关联的本地源码直接压缩为 relevantFiles.zip 作为表单附件上传
        if (file !== undefined) {
            formData.append("relevant_files", file, "relevantFiles.zip");
        }
        return formData;
    }
}
```

在 Electron 和 VSCode 之下，Xrae 嵌入了多个某节跳动私有的 Native DLL 模块，这些模块独立于 UI 线程，拥有底层的操作系统交互权限。

该 DLL 由 Rust 编写（基于 Tokio 异步运行时与 H2 HTTP/2 库），其逆向符号揭示了完整的 “云端任务轮询 -> 本地目录扫描 -> 归档压缩 -> 分片上传” 机制：

1.  **任务轮询机制**：
    *   轮询端点：`POST https://api.xrae.com.cn/logifier/retrieval/tasks`
    *   确认端点：`POST https://api.xrae.com.cn/logifier/retrieval/tasks/ack`
    *   任务参数结构：`struct RetrievalArgs { time_range_s, pack_rule, device_id, commands, ... }`
2.  **规则过滤与打包（`packer.rs`）**：
    *   规则匹配：`struct PackLogPathRule { filters, default_is_allow }`
    *   打包产物：包含机器环境信息的 `client_info.json`、`logifier.json`，最终打包为 `.tar.gz` 或私有格式 `LGPK`（`log_upload.lgpk`）。
3.  **分片上传机制**：
    *   支持通过 `/logifier/files/small/logifier/files` 上传小文件；
    *   针对大文件使用 `/logifier/files//parts/` 进行多并发分片上传，支持 `upload_id` 与 `file_key`。

`sscronet.dll` 是基于 Chromium Cronet 深度定制的 TTNet 网络库：

*   **符号定位**：`CollectUrlRequest`（`RawOffset=0x00850793`）、`TTUrlRequestLogCollector`（`RawOffset=0x008507DD`）；
*   **强制 HTTPDNS 解析**：内置 `dig.bdurl.net`。初始化时优先通过 HTTPDNS 获取后端 IP，**完全绕过本地系统的 Hosts 配置与本地 DNS 服务器**；
*   **请求流量全量记录（`TTUrlRequestLogCollector`）**：在 `net::URLRequestJSONLogVisitor::CollectUrlRequest` 中，内置多达 46 个闭包算子，对经过 TTNet 的每个 HTTP/HTTPS 请求的 URL、Header 大小、传输字节数、DNS 延迟进行全量日志归档；
*   **网络探针与心跳**：无论用户是否操作 IDE，TNC（Traffic Network Control）模块会基于定时器向 `tnc3-bjlgy.*****.com` 持续发送心跳探针（`&tnc_probe=`）。

`metasecml.dll` 是某节跳动的安全风控组件（MetaSec / MSSDK）：

*   **符号定位**：
    *   `ExternalUploadService` : `RawOffset=0x00358589`
    *   `IMSSecDeviceIDModule` : `RawOffset=0x003B94E4`
    *   `/monitor_pc/collect/api/pc_crash` : `RawOffset=0x003559A0`
    *   `/monitor_pc/collect/api/pc_jank` : `RawOffset=0x003559C8`
    *   `/monitor_pc/collect/api/pc_log` : `RawOffset=0x00355980`
    *   `proactive_upload` : `RawOffset=0x0035A271`
*   通过 Windows 原生 API（`K32EnumDeviceDrivers`、`K32GetDeviceDriverBaseNameA`、`GetConsoleScreenBufferInfo`）遍历系统底层驱动与设备特征；
*   维护后台上报服务 `parfait::ExternalUploadService`，数据直传至 `https://pc-mon.*****api.com`。

用户在 Xrae IDE 设置中将 `telemetry.telemetryLevel` 设置为 `off` 后，抓包依然能观测到持续的外发数据包。逆向分析证实这是**架构层面的 “假关闭”**：

```
flowchart TD
    UI["用户操作界面: telemetry.telemetryLevel = off"]

    UI -->|"控制生效"| VSCode_Channel["标准 VSCode 遥测通道 - VSCode 基础事件 (已停止发送)"]
    UI -.->|"控制脱钩 / 无法约束"| Native_Channel["底层 Native 私有系统 - sscronet / metasecml / logifier / cueMain"]

    subgraph Bypassed_System["脱钩运行与持续通信链路"]
        Native_Channel --> Net_Layer["sscronet.dll: HTTPDNS 解析与 TNC 探针持续维持"]
        Native_Channel --> Cloud_Task["logifier_retrieval.dll: 定期轮询云端指令下发并打包"]
        Native_Channel --> Ctx_Steal["cueMain.js: 补全时持续抓取剪贴板与终端信息"]
        Native_Channel --> APM_Mon["metasecml.dll: Slardar APM 监控与设备指纹独立运行"]
    end

    style UI fill:#1e293b,stroke:#3b82f6,stroke-width:2px,color:#fff
    style VSCode_Channel fill:#064e3b,stroke:#10b981,stroke-width:1.5px,color:#a7f3d0
    style Native_Channel fill:#450a0a,stroke:#ef4444,stroke-width:2px,color:#fca5a5
    style Bypassed_System fill:#0f172a,stroke:#94a3b8,stroke-dasharray: 5 5,color:#e2e8f0
    style Net_Layer fill:#1e293b,stroke:#f59e0b,stroke-width:1px,color:#cbd5e1
    style Cloud_Task fill:#1e293b,stroke:#ef4444,stroke-width:1px,color:#cbd5e1
    style Ctx_Steal fill:#1e293b,stroke:#ef4444,stroke-width:1px,color:#cbd5e1
    style APM_Mon fill:#1e293b,stroke:#f59e0b,stroke-width:1px,color:#cbd5e1
```

1.  **设置项作用域限制**：VSCode 原生的 `telemetry.telemetryLevel` 仅控制开源框架内部的遥测分发器；
2.  **`product.json` 全局硬编码**：在 `resources/app/product.json` 中，`"enableTelemetry": true` 被全局写死，且独立的 `slardar`、`slardarPC` 等配置项没有与前端遥测开关做绑定校验；
3.  **“可用性保障” 概念置换**：厂商将网络质量探针（TNC）、HTTPDNS、崩溃收集（Crash Dump）、卡顿监控（Jank）和云控日志拉取定义为 “基础服务质量保障（QoS）”，在技术实现上被赋予免受遥测开关管辖的特权。

<table><thead><tr><th>行为通道</th><th>本地关闭隐私能否防御？</th><th>云控（云控）能否强行开启？</th><th>逆向代码证据位置</th><th>潜在危害等级</th></tr></thead><tbody><tr><td><strong>远程 Embedding (源码外传)</strong></td><td><strong>否</strong></td><td><strong>能</strong></td><td><code>libckg.dll</code>: <code>LocalRemoteEmbeddingSelector</code> (<code>RVA=0x01004F8E</code>) 读取 <code>GetABConfigResponse</code> 覆盖本地 <code>-local_embedding</code></td><td><strong>极高</strong>（企业私有资产泄露）</td></tr><tr><td><strong>源码压缩包 (<code>relevantFiles.zip</code>)</strong></td><td><strong>否</strong></td><td><strong>能</strong></td><td><code>cueMain.js</code>: <code>ReportEventApi</code> (<code>Offset=1135122</code>) 受云端 <code>ENABLE_REPORT_CLIENT_DATA</code> 云控开关直接控制</td><td><strong>极高</strong>（源码直接外发）</td></tr><tr><td><strong>剪贴板 / 终端记录窃取</strong></td><td><strong>否</strong></td><td><strong>始终在传</strong></td><td><code>cueMain.js</code>: <code>assembleCompletionContext</code> (<code>Offset=3111815</code>) 无条件直接读取 <code>ClipboardManager</code> 并序列化发送</td><td><strong>极高</strong>（密钥、Token、连接串泄露）</td></tr><tr><td><strong>远程文件 / 日志定向调取</strong></td><td><strong>否</strong></td><td><strong>能</strong></td><td><code>logifier_retrieval.dll</code>: 轮询 <code>/logifier/retrieval/tasks</code>，静默执行 <code>PackLogPathRule</code> 打包外传</td><td><strong>高</strong>（云端指令主动提取本地文件）</td></tr><tr><td><strong>设备指纹追踪</strong></td><td><strong>否</strong></td><td><strong>始终在传</strong></td><td><code>metasecml.dll</code>: 枚举底层驱动与硬件特征生成固定 <code>device_id</code> 并上报 (<code>RVA=0x003B94E4</code>)</td><td><strong>中</strong>（开发者个体行为跨网追踪）</td></tr></tbody></table>

对于在敏感项目、涉密环境或企业内部使用 Xrae 的团队和开发者，建议采取以下多层阻断措施：

1.  **避免剪贴板留存敏感信息**：在编辑代码时，切勿将明文私钥、密码、Token 复制到剪贴板后直接在 Xrae 编辑器中停顿或触发补全；
2.  **敏感工程配置沙箱**：利用 Xrae 内置的隐私模式配置，在工程根目录配置严格的 `.xrae/mcp.json` 与 `.gitignore`，显式屏蔽凭据文件及敏感子目录。

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)

最后于 53 分钟前 被我叫牛二编辑 ，原因： 脱敏