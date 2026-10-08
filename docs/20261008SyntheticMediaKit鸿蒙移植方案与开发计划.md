# ArktsSyntheticMediaKit 移植方案与分阶段开发计划

> 日期：2026-10-08  
> 状态：待评审（仅设计，不实施）  
> 目标仓库：`kyinwind/ArktsSyntheticMediaKit`  
> 源仓库：`kyinwind/SyntheticMediaKit`（main；Swift Package 0.1.3）  
> 原则：业务语义对齐、鸿蒙原生实现、接口可扩展、合规能力如实报告。

## 1. 目标、边界与已核实事实

本工程把现有 Swift SyntheticMediaKit 的**已实现能力**移植到 HarmonyOS NEXT 的 ArkTS 可复用组件库，供鸿蒙应用集成。第一阶段不进行语音克隆模型推理：只负责录制、挑选及管理克隆所需的参考音频、授权记录、调用许可门禁，以及 AI 合成内容的音视频标识。目标是提供可独立集成的 HAR 和示例应用。

**源代码审计依据：**

- `Package.swift`：Swift 6、iOS 17+/macOS 14+；产品包括 SyntheticMediaCore、SyntheticMediaActivation、SyntheticMediaConsent、VoiceEnrollment、MediaLabeling、SyntheticMediaUI 和两个 CLI。
- `README.md`：当前 0.1.3；三段随机朗读挑战、明确选择参考录音、中英双语界面、授权撤回、音量调整、媒体标识。
- `VoiceRecordingController.swift`：16-bit / 24 kHz / mono Linear PCM，实时平均音量、暂停之外的启动/停止/取消和权限检查。
- `VoiceEnrollmentService.swift`：授权门禁、三段挑战、ASR 文本匹配、质量检查、失败冷却、手动选择主要参考音频、增益处理、恢复原件、删除数据；默认文本相似度阈值 0.82。
- `VoiceAudioAnalyzer.swift`：整体/活跃语音 RMS dBFS、峰值、静音比例、削波比例、噪声底、推荐增益；默认过小阈值 -26 dBFS、削波/过大峰值 -1 dBFS、目标活跃语音 -20 dBFS、最大建议增益 12 dB。
- `BasicAudioQualityChecker.swift`：2.5–15 秒时长、削波比例上限 0.02、静音比例上限 0.8；音量过小作为 warning 而非单独失败原因。
- `MediaLabeling/AVMetadataLabeler.swift`：QuickTime metadata description 中编码 JSON，keywords/creator，读取回验 contentID。
- `MediaLabeling/VideoVisibleLabeler.swift`：视频图层合成、位置/出现区间、音轨保留、导出、可选元数据。
- `AudioExplicitLabeler.swift`：音频前置或后置提示片段。
- `AudioWatermarkProvider.swift`：仅有 embed/detect 可插拔协议及 capability 描述，**无生产级嵌入/检测算法**。
- `README.md` 明确 C2PA 为未来扩展；原库没有承诺法律合规认证。
- 鸿蒙目标仓库目前仅有 `.gitignore`，需要建立 HarmonyOS 工程与 HAR 包结构。

**不在当前移植中实施：** Swift 源库业务逻辑变更、语音克隆引擎、云端人声身份核验、抗转码鲁棒数字水印算法、C2PA 签名实现；这些能力可单独评审立项。

## 2. 移植策略与功能矩阵

| 源模块 | ArkTS 模块 | 首版要求 | 适配方式 / 注意 |
|---|---|---|---|
| SyntheticMediaCore | core | 必须 | 模型、错误码、状态、contentID、存储抽象 |
| SyntheticMediaActivation | activation | 必须 | SHA-256 摘要、产品命名空间、能力解锁；绝不在包内存明文密钥 |
| SyntheticMediaConsent | consent | 必须 | 五项独立确认、政策/文案版本、同意时间/撤回时间 |
| VoiceEnrollment | voice | 必须 | 录音、三条随机提示、录音质量、ASR 依赖注入、文字匹配、冷却、优选与手选 |
| SyntheticMediaUI | ui | 必须 | ArkUI 完整入口、状态导航、中文和英文资源、权限、播放/重录/删除 |
| MediaLabeling 显式音频 | labeling/audio | 必须 | 提示音片段插入，前置/后置，重新导出并试听 |
| MediaLabeling 视频可见标识 | labeling/video | 必须（可能分阶段） | 指定文字、位置、时间范围、输出与可见性检查 |
| MediaLabeling 元数据 | labeling/metadata | 必须（可能分阶段） | 写入/读取/校验 AI 属性、服务提供者、内容 ID |
| MediaLabeling 检测扩展 | watermark | 预留 | Provider 注册/检测接口 + unavailable 状态；不虚报可用 |
| C2PA | c2pa | 预留 | 仅接口与明确 unavailable，不内置签名实现 |
| 两个 Swift CLI | tools / scripts | 后续可选 | 跨平台提示语校验与密钥生成脚本，不要求出现在 HAR 内 |

关键：**相同功能并不等于 Swift 与 ArkTS API 逐字对齐。** 对外使用对象/接口和错误码应保持语义稳定；文件 URI、异步模型、系统授权、音视频容器差异用适配器隔离。

## 3. 建议工程组织

```text
ArktsSyntheticMediaKit/
├─ docs/
│  └─ 20261008SyntheticMediaKit鸿蒙移植方案与开发计划.md
├─ synthetic-media-kit/                 # HAR library module
│  ├─ src/main/ets/
│  │  ├─ core/
│  │  ├─ activation/
│  │  ├─ consent/
│  │  ├─ voice/
│  │  │  ├─ enrollment/
│  │  │  ├─ recording/
│  │  │  ├─ quality/
│  │  │  ├─ prompts/
│  │  │  └─ storage/
│  │  ├─ labeling/
│  │  │  ├─ audio/
│  │  │  ├─ video/
│  │  │  ├─ metadata/
│  │  │  └─ watermark/
│  │  ├─ ui/
│  │  └─ index.ets
│  ├─ src/main/resources/
│  └─ oh-package.json5
├─ entry/                               # 可运行 ArkUI 演示工程
├─ tests/                               # 视实际 HarmonyOS 测试工具布局调整
├─ README.md
└─ oh-package.json5
```

实际根工程/模块配置遵循创建工程时所使用的 DevEco Studio / HarmonyOS SDK 模板，不建议提前手造未经编译验证的 build-profile 文件。HAR 本身不请求或自动授予权限；宿主必须在 module.json5 中声明并在运行时请求麦克风等权限。优先使用应用沙箱目录保存敏感录音。

## 4. 业务接口和生命周期

拟公开的主要对象（接口草案，待编译验证）：

- `SyntheticMediaKit.configure(config)`：注入 productID、版本、存储路径、ASR/权限/持久化适配器。
- `ConsentService.getState()/accept()/withdraw()`：必须持久保存政策版本和五项同意。
- `VoiceEnrollmentService.beginChallenge(locale, count=3)`：无有效同意时禁止开始；随机且不重复。
- `VoiceRecorder.requestPermission()/start()/stop()/cancel()/getLevel()`。
- `VoiceEnrollmentService.validate(itemId, audioUri)`：先质量指标，再调用宿主 ASR，最后文本相似度与失败计数。
- `VoiceEnrollmentService.getRecommendedRecordingId()/finalizeEnrollment(selectedId)`：推荐算法只辅助；不能自动代替用户选定。
- `VoiceEnrollmentService.selectPrimaryReference()/analyzeReferenceAudio()/applyRecommendedGain()/applyGain()/restoreOriginal()/deleteVoiceData()`。
- `VoiceCapabilityAuthorizer.authorize()`：激活策略（如启用）、同意状态、注册通过状态、参考文件存在性同时校验。
- `MediaLabeler.addAudioPrompt()/addVideoVisibleLabel()/writeMetadata()/verifyMetadata()`。
- `AudioWatermarkProvider.embed()/detect()`：在安装真实 provider 前返回 `unavailable`；绝不输出虚假的 success。

服务层应返回稳定的枚举和结构化错误码，而非依赖直接展示英文异常文本；所有耗时音视频操作暴露 Promise、进度、取消和资源释放语义。入参使用可访问的 URI/FD 适配层，不假设所有 HarmonyOS 文件都能当作普通路径处理。

注册状态：notStarted → recording → validating → ready；异常转为 failed / cooldown / needsReverification；同意撤回、政策版本变更、参考录音缺失或变更必须触发重新校验或拒绝授权。保证同一时间只有一个录音会话，进程终止后能够安全恢复或清理未完成的临时数据。

## 5. HarmonyOS 底层技术选型和验证事项

### 5.1 音频录制及音质

优先评估 `@kit.AudioKit` 的 AudioCapturer（原始 PCM 实时缓冲，适合自定义采样与电平分析）与 `@kit.MediaKit` 的 AVRecorder（编码录制较方便）。需求是**实际产出 24 kHz / mono / 16-bit PCM WAV**；不可因为底层某个采样率不支持就静默生成有误的 WAV header。先在目标真机检查采样率、声道、录制来源、状态转换及打断场景，必要时录制为支持的采样率再做确定性重采样并写正确 WAV 头。

音质算法优先纯 ArkTS：按 PCM 窗口计算 RMS、peak、静音、削波、活跃语音、噪声底，复刻源库阈值；建议增益 = min(目标电平差, 峰值余量, 最大增益)。处理一律从不可变主文件生成临时 WAV，检查无削波、长度与文件完整性后原子替换当前工作副本；保留 master 供还原；音频修订号随主要录音变更/增益变更递增。

ASR 需设 `SpeechTranscriber` 接口由宿主注入。**不应默认系统 ASR 离线可用**：先评估 HMS/系统语音识别能力、语言覆盖、是否需网络、鉴权及数据出境；没有可用适配器时明确反馈 unavailable，不跳过朗读匹配验证。

### 5.2 音频提示与视频显式标识

音频提示采用用户/宿主提供的标准提示音素材，实现开头或结尾拼接；统一采样率、声道及编码，再导出指定兼容容器。验证输出有提示、原音完整、时长合理。

视频标识采用固定画面文字与支持打开头/指定时间范围的可见叠加。HarmonyOS `AVTranscoder` 文档公开了 `addWatermark` 接口，但其具体支持版本、平台、静态图能力、透明度、坐标及动态显示区间**必须先真机 POC 核实**；不能假定与 Apple 的 AVVideoCompositionCoreAnimationTool 完全等价。若不支持动态区间或文字图层，研究可替代的视频逐帧渲染 + 编码/封装管线，或 C++/NDK 方案。移动端功耗、内存与长视频取消恢复都要专项测试。

### 5.3 元数据隐式标识

沿用 `SyntheticMediaMetadata` 语义：schemaVersion、isAIGenerated、providerName、providerCode、contentID、modelIdentifier、generatedAt、manifestReference。区分三类：

1. **文件元数据标识**：在目标容器真实嵌入字段并进行重新读取验证；要求导出仍可正常播放。
2. **数字水印**：修改媒体内容、期望在压缩/剪切后仍可检测；原 Swift 仅定义接口，不作为已交付功能。
3. **内容哈希/外部清单**：辅助留痕，不等同于已嵌入媒体的元数据或鲁棒水印。

优先验证 MP4/M4A 的可携带字段、持久性及互操作性（包含 Swift 原库的 `SyntheticMediaKit:` metadata JSON 格式）。不能可靠嵌入的文件格式必须返回失败或 unavailable，禁止写个同名 sidecar 后对外宣称已嵌入。生成文件完成后计算 SHA-256 摘要，报告每个阶段的真实状态。

### 5.4 安全与数据生命周期

- 录音是敏感数据；最小权限、沙箱存放、不得在日志中输出音频原文/密钥/可识别信息。
- 保留政策版本、明确同意和撤回时间；撤回后停止授权并删除音频、临时文件、主副本、参考状态。可保留最小撤回留痕，需在文案中说明。
- 读取持久化状态时做模式版本校验和损坏数据恢复；产品 ID 命名空间隔离。
- 加密存储根据风险评估考虑 HarmonyOS HUKS；不把假设的“系统自动加密”作为安全保证。
- 离线注册码采用与 Swift 兼容的归一化与 SHA-256（`productID:normalizedKey`）；密钥摘要比较防时间侧信道。设备/本地激活只作为访问控制，不等同于验证现实人物身份。
- 未授权录制、拒绝麦克风、接电话、中途退出、空间不足、后台终止、应用卸载等场景均设计恢复/清理路径。

## 6. 合规边界（必须在开发评审时确认）

《人工智能生成合成内容标识办法》（2025-09-01 施行）明确区分显式与隐式标识；隐式标识要求在生成合成内容文件的元数据中记录生成合成属性、服务提供者、内容编号等信息，并鼓励数字水印。仅“加入一个文件旁边的 JSON”不能替代嵌入元数据，也不能把可见角标当成隐式标识。

法规原文：https://www.cac.gov.cn/2025-03/14/c_1743654684782215.htm

在发布前需要逐项核对有关强制性标准、平台要求和适用服务场景；本库提供技术手段，**不自动证明宿主产品合法合规**。音视频导出链路按业务场景判断显式标识应位于媒体本体还是交互界面；支持导出时优先保证文件自身包含所需标识。对视频首帧标识与全程叠加的匹配程度，需要形成合规验收清单。不得通过声纹/录音自动断言真实身份或合法授权，宿主仍需处理个人信息告知、同意与滥用防护。

## 7. 开发里程碑、产物和验收

| 阶段 | 主要任务 | 交付件 | 验收门槛 |
|---|---|---|---|
| P0 基线/风险验证 | 固定 Swift 源基线；建立 HAR+Demo；实测录音格式、音视频封装、元数据读写、视频叠加和 ASR 可用性 | 能运行的空库+实验报告+功能支持矩阵 | 每个关键 API 有真机结果，不支持项明确替代路线 |
| P1 Core/Activation/Consent | 数据模型、存储、摘要、授权记录、撤回、门禁 | 可单元测试的 ArkTS 核心模块 | 产品隔离、状态迁移、密钥格式对齐原库 |
| P2 录音质量与注册 | 麦克风权限、WAV 录制、PCM 指标、随机挑战、ASR 插件、失败冷却、优选与手选、音量调节和删除 | VoiceEnrollment 服务及测试数据 | 三条挑战通过后才可使用；手选生效；回滚及删除完整 |
| P3 ArkUI 全流程 | 中英文文案、五项确认、入口/录制/音量/播放/重录/选主样本/撤回界面 | 可交互 Demo | 正常与异常路径可从 UI 走通 |
| P4 媒体标识 | 音频提示拼接、视频可见标识、元数据读写/回验、SHA-256 报告 | MediaLabeling HAR API + 样例媒体 | 导出可播放；视频肉眼可见；元数据回读 contentID；失败可诊断 |
| P5 稳定性与分发 | 性能/崩溃/音频中断/大文件/权限/兼容性/安全审计、接入说明、版本发布 | 可复用 HAR、Demo、README、API 文档、变更记录 | 目标机型 smoke tests 通过、阻断级问题归零 |

**顺序门槛：** P0 不通过时不得把 P4 的动态视频水印/元数据标成“已完成”；可先交付 P1–P3 并报告缺口。P2 在没有 ASR provider 时可做录制/质量检测 Demo，但**不得声称朗读核验合格**。P4 的音视频元数据验证必须覆盖文件导出后的实际文件。

### 测试矩阵

- **单元测试**：同意/撤回/过期政策、密钥归一化/摘要一致性、文本相似度、随机提示无重复、音质极值、冷却超时、增益边界、状态迁移、序列化兼容。
- **音频夹具**：正常人声、低音量、过大/削波、静音、过短/过长、不同采样率和损坏的 WAV；对比 Swift 结果，以误差容限定义一致性。
- **集成测试**：真实麦克风权限、录音中断、前后台、连续录音、关闭 App 再进入、主文件恢复、删除与撤回。
- **媒体测试**：常见 MP4/M4A 样本、横屏/竖屏、有无音轨、不同编码、短长片段、转码后显式标识与 metadata 是否存在。
- **水印未来扩展测试**：压缩、重采样、裁切、噪声、误检率/置信度；只对实际加载的 provider 执行并报告，不将协议测试计作算法达标。
- **兼容性**：以最终选定的 HarmonyOS API Level 和手机/平板机型为准；文档化硬件和 ROM 差异。

## 8. 需用户评审的决策

1. **最低 HarmonyOS API Level**：建议先按 API 12+ 的功能风险建立 POC，最终由目标设备/商店要求决定；不能只凭 API 存在就判断所有机型支持。
2. **目标端**：手机优先，平板次之；如需 PC/2in1，媒体能力应单独验证。
3. **ASR 实现**：库仅定义注入接口，还是同时提供一个 HarmonyOS 在线/离线 ASR 适配实现？建议先走注入接口。
4. **视频标签范围**：首版必须复制 Swift 的“指定时间段出现”，还是允许首版全程叠加、下一版扩展？建议先完成 POC 再定。
5. **元数据容器**：首版主攻 MP4/M4A；其它格式按实际可验证能力增补。
6. **分发方式**：仓库内 HAR/源码集成先行；是否上架 OpenHarmony 三方包仓库或企业私有包源另行确认。
7. **授权/克隆风险**：用户确认“自己的声音”不等同于可信身份验证。是否要求额外的活体、账号或人工审核，属于宿主侧策略决定。

## 9. 阶段完成判定与禁止事项

只有当 API 可调用、自动化测试通过、真机输出已验证、Demo 完整走通、文档描述与行为一致时，才将模块标记为已实现。严禁：

- 将 Swift 源码复制为 ArkTS 而不做音视频适配；
- 绕过质量或 ASR 核验直接进入 ready；
- 在录音/增益处理中丢失原版文件；
- “用户未选择”时自动设定主参考录音；
- 把元数据、sidecar 文件和抗转码数字水印混为一谈；
- 未做嵌入回读校验就宣称隐式标识成功；
- 用“框架有接口”冒充“符合监管要求”；
- 未经本方案评审就开始批量功能开发。

## 10. 下一步执行协议

本次提交仅包含该设计文档，**不改动任何 ArkTS 功能代码**。评审通过后先实施 P0 并提交开发 POC 和验证矩阵，再依 P1 → P5 的顺序逐阶段开发。后续每阶段先提交代码、测试和变更说明，再判定进入下一阶段。

## 参考资料

- Swift 源库（private）：https://github.com/kyinwind/SyntheticMediaKit
- 源库 README、Package.swift、Sources/VoiceEnrollment、Sources/MediaLabeling、Sources/SyntheticMediaConsent 等实测读取文件。
- OpenHarmony AVRecorder 指南：https://github.com/openharmony/docs/blob/master/en/application-dev/media/media/using-avrecorder-for-recording.md
- HarmonyOS AVTranscoder API：https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-media-avtranscoder
- 《人工智能生成合成内容标识办法》：https://www.cac.gov.cn/2025-03/14/c_1743654684782215.htm
