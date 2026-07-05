+++
title = "Unity中的BeatTracking算法"
date = "2026-07-05- 17:28:00 +0800"
tags = ["Game Design", "technology"]
categories = ["technology"]
slug = "Beat tracking algorithm in Unity"
indent = false
# draft = true
# dropCap = false
# katex = true
+++



在开发节奏游戏（Rhythm Games）或音频同步的可视化项目时，**自动节拍追踪（Beat Tracking）与BPM检测**是一个非常核心的技术点。然而，当我们直接在 Unity 中运行一些经典的开源算法（如 `UniBpmAnalyzer`）时，往往会发现理论与实际运行之间存在巨大的鸿沟。

从经典的“双倍频误差”到“高响度电音的全面失效”，我们在构建一个高泛化性的 Unity 节奏追踪引擎时，经历了一场长达数轮的渐进式架构迭代。本文将详细记录我们在开发过程中遇到的物理与数学痛点、走过的弯路、失败实现的分析以及最终落地的一套自适应双轨对齐方案。

---

## 一、 经典痛点：倍频误差（Octave Error）与初步尝试

作为起点，`UniBpmAnalyzer` 采用的是经典的音频信号分析机制：通过对音频进行采样、切帧，计算出音量能量的微分，然后将其与特定频率的正弦/余弦波进行乘积求和（类似离散傅里叶变换的滑动匹配），寻找相关度最高（Match Score 最大）的周期作为 BPM。

### 1. 为什么会自动检测出双倍 BPM？
在实际测试中，我们经常遇到 **“倍频误差（Tempo Doubling / Octave Error）”**。例如，一首 120 BPM 的歌，算法会判定为 240 BPM。
* **数学重合性**：如果音频在 120 BPM（每秒 2 拍）处有强烈的能量起伏，它在 240 BPM（每秒 4 拍，即 8 分音符）的理论网格上也必然是对齐的。
* **切分音与高频噪声的干扰**：现代流行乐和电音中充斥着大量的弱拍（Hi-hat 镲片、Shaker 沙锤）以及第 2、4 拍的军鼓（Snare）。在计算全频包络时，算法极易把这些中高频的次级振幅判定为有效节拍点，从而使 240 BPM 处的余弦匹配得分超越了 120 BPM。

### 2. 第一轮渐进优化
为了抑制倍频，我们尝试在算法各阶段引入干预机制：

* **方案A：高斯偏好权重（Tempo Prior）**
  DJ 软件（如 Rekordbox）为了防止倍频，会在计算匹配得分时加入人类听觉偏好的高斯分布权重（Gaussian Prior），将权重中心设在人类听觉最舒适的 120 BPM 附近：
  $$\text{Weight} = \exp\left(-\frac{(\text{bpm} - \mu)^2}{2\sigma^2}\right)$$
  在循环计算 Match 评分时乘以该系数，能够对超出常规的极高 BPM（如 240+）进行数学抑制。

* **方案B：后处理折半校验（Post-processing Halving）**
  当算法选择出最终的最大匹配点 $BPM_{\text{raw}}$ 后，我们不直接信任它。如果该值大于 140，我们会回头去核对它一半（或三分之一）位置的匹配度。如果 $BPM_{\text{half}}$ 的能量得分达到了最大得分的 70% 以上，由于人类本能更倾向于跟从慢速的重拍（Downbeats），我们将主动将其修正为折半后的 BPM。

* **方案C：一阶 RC 低通滤波器（Low-Pass Filter）**
  既然决定歌曲重拍的主要是底鼓（Kick）和低音贝斯（Bass），我们尝试在调用 `CreateVolumeArray` 前，在 C# 中对读取出来的原始 Audio Sample 进行极简单的低通滤波预处理（滤除 150Hz 以上的所有高频信息）：
  ```csharp
  // 简单一阶RC差分方程
  float dt = 1f / sampleRate;
  float RC = 1f / (2f * Mathf.PI * cutoffFrequency);
  float alpha = dt / (RC + dt);
  samples[i] = previousValue + alpha * (currentValue - previousValue);
  ```
  在 BPM 搜索端引入该低通滤波后，倍频误差得到了明显的改善。

---

## 二、 动态重音（Onset）的引入与无声留白处理

完成了 BPM 精确提取后，我们要产生节拍的时间戳。如果直接使用“固定时间计时器（Metronome）”累加时间戳，在面对复杂的真实音乐时会面临失效。

### 1. 为什么固定节拍器无法应对真实场景？
* **前后无声剪裁**：歌曲前后通常会有数秒的完全静音或极弱的留白，若不剪裁，固定节拍器会从零秒开始空放，导致后续与鼓点完全错位。
* **乐曲中的静音/留白阶段**：在一些歌曲的中途过渡段（Bridge）或歇拍段，根本没有鼓点。若固定节拍器继续输出，玩家将会在无声区尴尬地点击。

### 2. 构建 Onset（瞬态能量起音）提取系统
为了捕捉物理上的真实重音，我们编写了基于滑动的 **动态能量阈值法（Dynamic Energy Threshold）**。其核心逻辑是将整首歌拆解为更精细的能量块（Block，1024 采样点约 23ms），计算每个块的瞬时 RMS 能量：
$$E = \sqrt{\frac{1}{N} \sum_{n=0}^{N-1} x^2[n]}$$
然后比对该帧能量与局部周围窗口（前 20 帧与后 20 帧，双向约 1 秒）的平均能量。当 $E_{\text{instant}} > E_{\text{average}} \times \text{Multiplier}$ 时，判定发生了一次物理鼓点（Onset）。

为了防爆音，我们对首尾静音区采用了双向扫描截断算法（`TrimSilence`），并在裁剪边缘施加极短的音量线性渐变（Fade In/Out），防止截断处空气压力突变导致扬声器发出“啪”的爆音（Pop Click）。

---

## 三、 【弯路与思考】双轨对齐的锁相环（PLL）重置陷阱

此时，我们手上有两个数据：
1. **BPM 理论网格**（刚性、均匀，但无法自动对齐鼓点相位）。
2. **物理 Onset 时间戳列表**（有机、真实，但由于演奏误差和噪音可能产生丢包或多包）。

为了融合两者，我们最初尝试设计一个**动态锁相重置逻辑**（类似电学中的锁相环 PLL）：
```csharp
// 【失败的尝试代码片段】
if (minDiff <= tolerance) {
    alignedBeats.Add(closestOnset);
    // 试图动态重置下一次网格的计算起点，以此“呼吸”适应音乐
    currentGridTime = closestOnset + beatInterval; 
}
```

### 1. 为什么“动态重置起点”是一个灾难性的实现？
在进行小提琴、钢琴等真人乐器伴奏或非严谨电子混音测试时，这一行代码引发了极其不稳定的节拍重构（Jitter）：
1. **相位被杂音/切分音带偏**：在某些小节，鼓手敲了一个八分音符的切分音。动态对齐引擎会误认为网格相位发生了偏移，直接把网格原点向后平移了半拍。
2. **多米诺骨牌效应**：一旦当前网格被带偏，由于下一次网格是基于 `closestOnset` 计算的，这种平移误差会被不断向后传递和累积。原本十分稳固的 4/4 拍音乐，在后半段全部变成了杂乱无章的抖动。
3. **节奏游戏手感毁灭**：两个连续节拍的时间差可能在 `0.46s` 和 `0.30s` 之间来回跳跃，产生严重的“马蹄声”，玩家无法通过肌肉记忆进行稳定的预判。

### 2. 溯源思考：骨架必须是刚性的
通过这次失败，我们得出了一个重要的架构认知：
**在节奏游戏开发中，我们不能期望网格在局部随物理 Onset 的微小偏差而频繁位移。网格应当是一根极难弯曲的刚性铁轨（BPM Skeleton），物理 Onset 只是我们在铁轨上安装的减震垫（Onset Muscles）。**

为了解决累计漂移，我们必须使用完全等距的刚性步进：`currentGridTime += beatInterval`。而为了保证不漂移，对网格的 BPM 精度要求就上升到了浮点级。

---

## 四、 小数级高精度 BPM 与全局相位拟合（Global Phase Solver）

### 1. 小数级 BPM 的数学必要性
为什么整数 BPM 无法作为刚性骨架？
假设一首歌的真实 BPM 是 128.01。如果我们使用整数 `128` 来生成理论网格：
* 单拍误差为：
  $$\Delta T = \frac{60}{128} - \frac{60}{128.01} \approx 0.46875 - 0.46871 = 0.00004 \text{ 秒}$$
* 看似微乎其微。但当歌曲播放到第 150 拍（约 1 分多钟）时，由于是完全等距刚性累加，累计误差将达到：
  $$150 \times 0.00004 = 0.006 \text{ 秒}$$
* 此时若加上鼓手演奏微弱的延迟或混音延迟，误差极易突破容差窗口，迫使对齐算法判定为“冲突（Case C）”，从而全部走退落补正，失去了微调卡点（Case A）的意义。

在 `UniBpmAnalyzerHelper` 中，我们通过双阶段搜寻彻底实现了 `0.01` 步长的小数 BPM 精密检索。

### 2. 全局相位拟合（Global Phase Solver）的建立
有了精确的小数级 BPM 后，下一个问题是：**网格的第一拍应该从哪一毫秒开始？** 

如果直接锚定 `onsets[0]`，极易因为前奏的弱拍起音、淡入过渡音，使整首歌的网格整体向后平移。
为此，我们抛弃了局部锚定，实现了 **全局滑动相位拟合算法**：

1. 保持高精度 BPM 网格间距（如 `0.46871s`）绝对不变。
2. 假设起点候选值 $Offset$ 从 $0.0s$ 渐变到 $beatInterval$。我们以 **5毫秒（0.005s）** 为步长滑动扫描。
3. 对每个候选起点，向整首歌投影一个理论等距网格，并计算落入容差范围内的 Onsets 数量。
4. 我们编写了 $O(N+M)$ 的高效双指针比对函数：
   ```csharp
   private int CountGridMatches(float offset, float beatInterval, List<float> onsets, float duration, float tolerance)
   {
       int matches = 0;
       float gridTime = offset;
       int onsetIndex = 0;
       while (gridTime < duration) {
           // 快速越过容差窗左侧的 Onsets
           while (onsetIndex < onsets.Count && onsets[onsetIndex] < gridTime - tolerance) {
               onsetIndex++;
           }
           if (onsetIndex < onsets.Count && Mathf.Abs(onsets[onsetIndex] - gridTime) <= tolerance) {
               matches++;
           }
           gridTime += beatInterval;
       }
       return matches;
   }
   ```
5. **选择能让整首歌产生最多吻合拍数的那个滑动起点作为全局最佳 Offset。**

通过这个滑动拟合器，即使歌曲前奏极其吵闹，网格也能依靠强大的“全局投票机制”，强力、精准地卡死在整首歌最规律、最密集的鼓点集群上。

---

## 五、 泛化性的终极考验：大响度电音与“砖墙式”压限的处理

当我们满怀信心地将这套“高精度浮点 BPM + 全局相位拟合器”应用于测试 MDK 与 Ronald Jenkees 的名曲 **《Cyber Fight》** 时，系统给出了令人无法接受的检测数据：
> *[Grid-Lock Engine] 精准BPM: 128.01 | 全局最佳Offset: 0.528s。吻合对齐数: 40 | 理论补正数: 474 | 最终吻合率: 7.8%*

精准的 128.01 BPM 下，居然有高达 474 次补正，吻合率只有可怜的 7.8%！

### 1. 致命的“砖墙式”压限（Limiter / Brickwall Compression）
《Cyber Fight》是一首重型 Electro / Dubstep 电音。这类现代高能量舞曲为了追求极致的宏观听感响度，母带后期会经过极为激进的限制器（Limiter）处理。

这导致了两个物理死穴：
* **动态范围极其狭窄**：波形在视觉上被死死地压平，宛如一块没有波澜的红砖（Brickwall）。
* **RMS 比例法彻底失效**：原本使用的 `E_instant > E_average * Multiplier` 在这种音乐中失效了。因为合成器和 Sub-bass 持续高能轰鸣，导致背景滑动均值 $E_{\text{average}}$ 无时无刻都处于高位。当底鼓砸下时，它产生的相对增幅甚至不到 10%，根本无法跨越原本设定的 `1.35` 倍门限。

### 2. 回退全频分析导致的“半频旋律干扰”
我们曾尝试在分析 Onset 时去掉低通滤波器。但这引发了更大的灾难：去掉滤波后，BPM 分析端直接被高亮、交替的合成器高频旋律（Synth Leads）带偏，错误地算出了 **64.01 BPM**（旋律的行进周期），吻合率跌至 8.6%。

这让我们深刻地意识到：**BPM 周期追踪与 Onset 起音点追踪，在频带过滤策略上必须解耦！**
* **BPM 骨架检测**：必须保留 `150Hz` 窄带低通滤波，确保周期纯净度，过滤中高频旋律的二分频干扰。
* **Onset 肌肉检测**：不能一刀切。我们必须设计一个**双频带滤波器组**（Bass Band 与 Mid-High Band），同时捕获低频底鼓和中高频的军鼓与拍手。

### 3. 一阶正向差分与局部极大值检测（Local Peak Picking）
为了彻底解决现代高响度电音的“漏检”问题，我们将绝对能量比例法，重构为**基于一阶正向差分包络的斜率检测**：
* **一阶正向差分**：
  $$D[i] = \max(E[i] - E[i-1], 0)$$
  不看音量绝对值，而是比对“当前块能量相比于前一时刻能量的上涨速度（斜率）”。即使整首歌被限制器压得平整如镜，但在底鼓/军鼓击下去的微小瞬间，音量包络依然会有一个极快、极陡峭的**上冲瞬间**。这个“启动动作（Attack）”在差分 $D[i]$ 上会呈现出尖锐的脉冲峰值。
* **局部极大值过滤（Local Peak Picking）**：
  我们要求当前斜率必须满足局部峰值条件：
  $$D[i] > D[i-1] \quad \text{and} \quad D[i] > D[i+1]$$
  这保证了节拍只触发在波形开始上冲的那一瞬间，彻底消除了重音重叠和高频重复触发。

此外，由于双频带巴特沃斯滤波器在滤频时会造成能量信号的物理衰减，原有的静音门限 `minimumAbsoluteEnergy = 0.015f` 会将衰减后的电音重音误判为静音区过滤掉，这也是导致 7.8% 极低吻合率的直接元凶。我们将该阈值默认降为了 `0.005f`。

---

## 六、 架构落地：更具确定性的 Grid-Lock 对齐引擎

经历了上面的探索与试错后，我们在 Unity 中最终落地了这一整套高鲁棒性的 **`DynamicGridBeatSystem`**。

脚本内置了：
1. **空白裁剪与防爆音淡入淡出（Trim & Fade）**。
2. **窄带滤波高精度浮点 BPM 搜索（双阶段，0.01 精度）**。
3. **二阶巴特沃斯 IIR 双频分带滤波组**。
4. **一阶正向差分斜率斜率检测 + 局部极大值筛选**。
5. **基于 $O(N+M)$ 双指针的全局最佳相位拟合滑动器**。
6. **对齐冲突处理开关（Conflict Resolution Mode）与漏检诊断器**。

### 完整的 `DynamicGridBeatSystem.cs` 代码

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using UnityEngine;

public class DynamicGridBeatSystem : MonoBehaviour
{
    public enum ConflictResolution
    {
        UseBPMGrid,       // 冲突时：优先使用BPM理论骨架补正（死守刚性等距网格，最稳定、不抖动）
        UseEnergyOnset    // 冲突时：优先使用实际能量重音点（更偏向物理波形，容差大时易产生抖动）
    }

    [Header("Components")]
    [SerializeField] private AudioSource audioSource;
    [SerializeField] private AudioClip rawAudioClip;

    [Header("Trim Settings")]
    [Tooltip("裁剪空白的振幅门限（绝对值 0~1），低于此值视为空白")]
    [SerializeField] private float trimThreshold = 0.01f;
    [Tooltip("淡入淡出时长（秒），防止裁剪边缘产生爆音")]
    [SerializeField] private float crossfadeDuration = 0.01f;

    [Header("Beat Detection Settings")]
    [Tooltip("低通滤波器截止频率（Hz），保留底鼓/贝斯等节奏基底")]
    [SerializeField] private float lpfCutoff = 150f;
    [Tooltip("动态能量阈值系数。当前能量超过局部平均能量的多少倍时判定为能量重音（通量）")]
    [SerializeField] private float thresholdMultiplier = 1.35f;
    [Tooltip("绝对最小能量门限。由于滤波器衰减，建议降为 0.005f 以下，否则极易漏检")]
    [SerializeField] private float minimumAbsoluteEnergy = 0.005f;
    [Tooltip("触发冷却时间（秒），防止同一个鼓点高频重复触发")]
    [SerializeField] private float refractoryPeriod = 0.25f;

    [Header("Grid-Lock Settings")]
    [Tooltip("微调对齐的绝对时间容差（单位：秒）。追求网格绝对稳定均匀时建议收窄到 0.03 ~ 0.05")]
    [SerializeField] private float alignmentTolerance = 0.05f;

    [Header("Conflict Resolution")]
    [Tooltip("当理论BPM网格点与能量重音偏差超过容差（发生冲突）时，选择哪种纠错策略？")]
    [SerializeField] private ConflictResolution conflictResolution = ConflictResolution.UseBPMGrid;

    // 最终融合后的黄金节拍时间戳（秒）
    private List<float> finalBeatTimestamps = new List<float>();
    private int currentBeatIndex = 0;
    private AudioClip trimmedClip;

    // 节拍触发回调
    public Action<float> OnBeatTriggered;

    private void Start()
    {
        if (rawAudioClip == null || audioSource == null)
        {
            Debug.LogError("请关联 AudioSource 和原始 AudioClip！");
            return;
        }

        ProcessAndAlignBeats();
    }

    private void Update()
    {
        if (audioSource == null || !audioSource.isPlaying || finalBeatTimestamps.Count == 0) return;

        float currentTime = audioSource.time;

        // 顺序比对播放时间，精准跨越触发
        while (currentBeatIndex < finalBeatTimestamps.Count && currentTime >= finalBeatTimestamps[currentBeatIndex])
        {
            OnBeatTriggered?.Invoke(finalBeatTimestamps[currentBeatIndex]);
            currentBeatIndex++;
        }
    }

    public void ProcessAndAlignBeats()
    {
        // 1. 首尾空白裁剪，防爆音
        trimmedClip = TrimSilence(rawAudioClip, trimThreshold, crossfadeDuration);
        audioSource.clip = trimmedClip;

        // 获取裁剪后的纯净采样数据
        float[] samplesForAnalysis = new float[trimmedClip.samples * trimmedClip.channels];
        trimmedClip.GetData(samplesForAnalysis, 0);

        // 备份一份未处理的原始数据用于绝对静音判断
        float[] rawSamplesBackup = new float[samplesForAnalysis.Length];
        Array.Copy(samplesForAnalysis, rawSamplesBackup, samplesForAnalysis.Length);

        // 2. 基础骨架分析：窄带 LPF 预滤波（直接传递面板上的 lpfCutoff 变量）
        float baseBpm = UniBpmAnalyzerHelper.AnalyzeBpm(trimmedClip, lpfCutoff);
        if (baseBpm <= 0)
        {
            Debug.LogError("BPM 分析失败，无法建立基础网格骨架。");
            return;
        }

        // 3. 瞬态肌肉分析：应用高效二阶双频带滤波器组 (分出低频和中高频，融合检测)
        List<float> rawOnsets = AnalyzeRawOnsetsDualBand(samplesForAnalysis, trimmedClip.channels, trimmedClip.frequency);

        // 4. 双轨对齐与纠错引擎 (刚性小数网格模式 + 全局相位拟合)
        finalBeatTimestamps = AlignGridWithOnsets(baseBpm, rawOnsets, rawSamplesBackup, trimmedClip.channels, trimmedClip.frequency, trimmedClip.length);

        // 重置状态并播放
        currentBeatIndex = 0;
        audioSource.Play();
    }

    #region 刚性小数网格与全局相位拟合算法

    private List<float> AlignGridWithOnsets(float bpm, List<float> onsets, float[] rawSamples, int channels, int sampleRate, float duration)
    {
        List<float> alignedBeats = new List<float>();
        
        // 【退落安全保护】：如果物理重音一个都没检测到，退落为纯理论等距网格，保证游戏可玩
        if (onsets.Count == 0)
        {
            Debug.LogWarning("[Grid-Lock] 未探测到任何有效物理重音(Onset)。已自动回落为 100% 理论刚性网格。");
            float interval = 60f / bpm;
            float grid = 0f;
            while (grid < duration)
            {
                alignedBeats.Add(grid);
                grid += interval;
            }
            return alignedBeats;
        }

        Debug.Log($"[Debug] 自适应双频带探测器一共捕获到 {onsets.Count} 个物理节奏重音。");

        // 全局网格相位滑动检索
        float optimalOffset = FindOptimalOffset(bpm, onsets, duration);
        float beatInterval = 60f / bpm;

        float currentGridTime = optimalOffset;
        int alignedCount = 0;
        int correctedCount = 0;

        while (currentGridTime < duration)
        {
            float closestOnset = -1f;
            float minDiff = float.MaxValue;

            for (int i = 0; i < onsets.Count; i++)
            {
                float diff = Mathf.Abs(onsets[i] - currentGridTime);
                if (diff < minDiff)
                {
                    minDiff = diff;
                    closestOnset = onsets[i];
                }
            }

            // 【情况 A：基本吻合 (Aligned)】
            if (minDiff <= alignmentTolerance)
            {
                alignedBeats.Add(closestOnset); // 采信物理微调的时间点，实现极致卡点
                alignedCount++;
            }
            // 【偏离容差区间（发生冲突 或 进入无声过渡区）】
            else
            {
                bool isSilent = IsTimeRangeSilent(rawSamples, channels, sampleRate, currentGridTime, 0.15f);

                if (isSilent)
                {
                    // 【情况 B：静音留白】
                    // 静音段不写任何节拍，挂起
                }
                else
                {
                    // 【情况 C：检测到冲突！】
                    if (conflictResolution == ConflictResolution.UseBPMGrid)
                    {
                        // 优先使用 BPM 理论网格（保持节奏绝对均匀，不发生微移抖动）
                        alignedBeats.Add(currentGridTime);
                        correctedCount++;
                    }
                    else
                    {
                        // 优先使用实际能量重音点。
                        // 为了防止该 Onset 之前或之后已被其他相邻网格点占用，引入防重占用检测：
                        if (!alignedBeats.Contains(closestOnset))
                        {
                            alignedBeats.Add(closestOnset);
                            correctedCount++;
                        }
                        else
                        {
                            // 若此重音已被占用，自动安全退落为 BPM 理论网格，保证时间轴不出现重叠
                            alignedBeats.Add(currentGridTime);
                            correctedCount++;
                        }
                    }
                }
            }

            // 刚性等距递增
            currentGridTime += beatInterval;
        }

        float totalBeats = alignedCount + correctedCount;
        float rate = totalBeats > 0 ? ((float)alignedCount / totalBeats) * 100f : 0f;
        Debug.Log($"<color=cyan>[Grid-Lock Engine] 精准浮点BPM: {bpm:F2} | 全局最佳Offset: {optimalOffset:F3}s。吻合对齐数: {alignedCount} | 理论补正数: {correctedCount} | 最终吻合率: {rate:F1}%</color>");
        
        // 智能诊断：如果检测到的 Onset 太少，提示用户调整参数
        if (onsets.Count < totalBeats * 0.4f)
        {
            Debug.LogWarning($"<color=orange>[Grid-Lock Diagnostic] 检测到的物理节奏数({onsets.Count})显著少于期望的总拍数({totalBeats})。这通常是因为 minimumAbsoluteEnergy 在滤波器衰减后过大，导致了漏检。请尝试在 Unity Inspector 面板中将 Minimum Absolute Energy 调小至 0.005 或 0.002。</color>");
        }

        return alignedBeats;
    }

    private float FindOptimalOffset(float bpm, List<float> onsets, float duration)
    {
        float beatInterval = 60f / bpm;
        float bestOffset = onsets[0];
        int maxMatches = -1;

        float startSearch = Mathf.Max(0f, onsets[0] - beatInterval);
        float endSearch = onsets[0] + beatInterval * 2f;
        float stepSize = 0.005f;

        for (float offsetCandidate = startSearch; offsetCandidate <= endSearch; offsetCandidate += stepSize)
        {
            int currentMatches = CountGridMatches(offsetCandidate, beatInterval, onsets, duration, alignmentTolerance);

            if (currentMatches > maxMatches)
            {
                maxMatches = currentMatches;
                bestOffset = offsetCandidate;
            }
        }

        return bestOffset;
    }

    private int CountGridMatches(float offset, float beatInterval, List<float> onsets, float duration, float tolerance)
    {
        int matches = 0;
        float gridTime = offset;
        int onsetIndex = 0;

        while (gridTime < duration)
        {
            while (onsetIndex < onsets.Count && onsets[onsetIndex] < gridTime - tolerance)
            {
                onsetIndex++;
            }

            if (onsetIndex < onsets.Count && Mathf.Abs(onsets[onsetIndex] - gridTime) <= tolerance)
            {
                matches++;
            }

            gridTime += beatInterval;
        }

        return matches;
    }

    private bool IsTimeRangeSilent(float[] rawSamples, int channels, int sampleRate, float timeInSeconds, float windowSec)
    {
        int centerSample = Mathf.RoundToInt(timeInSeconds * sampleRate * channels);
        int windowSamples = Mathf.RoundToInt(windowSec * sampleRate * channels);

        int start = Mathf.Max(0, centerSample - windowSamples / 2);
        int end = Mathf.Min(rawSamples.Length, centerSample + windowSamples / 2);

        float sum = 0f;
        int count = 0;
        for (int i = start; i < end; i++)
        {
            sum += rawSamples[i] * rawSamples[i];
            count++;
        }

        if (count == 0) return true;
        float rms = Mathf.Sqrt(sum / count);
        return rms < minimumAbsoluteEnergy;
    }

    #endregion

    #region 音频裁剪、巴特沃斯滤波与一阶差分 Onset 检测

    public class ButterworthFilter
    {
        public enum PassType { Lowpass, Highpass }

        private float v0, v1; 
        private float b0, b1, b2, a1, a2;

        public ButterworthFilter(float frequency, int sampleRate, PassType passType)
        {
            float resonance = Mathf.Sqrt(2.0f); 
            float w0 = 2.0f * Mathf.PI * frequency / sampleRate;
            float alpha = Mathf.Sin(w0) / resonance;
            float cosw0 = Mathf.Cos(w0);

            float a0 = 1.0f + alpha;
            if (passType == PassType.Lowpass)
            {
                b0 = ((1.0f - cosw0) / 2.0f) / a0;
                b1 = (1.0f - cosw0) / a0;
                b2 = ((1.0f - cosw0) / 2.0f) / a0;
            }
            else 
            {
                b0 = ((1.0f + cosw0) / 2.0f) / a0;
                b1 = -(1.0f + cosw0) / a0;
                b2 = ((1.0f + cosw0) / 2.0f) / a0;
            }
            a1 = (-2.0f * cosw0) / a0;
            a2 = (1.0f - alpha) / a0;
        }

        public float Process(float input)
        {
            float output = b0 * input + v0;
            v0 = b1 * input - a1 * output + v1;
            v1 = b2 * input - a2 * output;
            return output;
        }
    }

    private List<float> AnalyzeRawOnsetsDualBand(float[] rawSamples, int channels, int sampleRate)
    {
        List<float> detectedOnsets = new List<float>();
        int blockSize = 1024 * channels;
        int totalBlocks = rawSamples.Length / blockSize;

        ButterworthFilter[] lowFilters = new ButterworthFilter[channels];
        ButterworthFilter[] highFilters = new ButterworthFilter[channels];
        for (int c = 0; c < channels; c++)
        {
            lowFilters[c] = new ButterworthFilter(200f, sampleRate, ButterworthFilter.PassType.Lowpass);
            highFilters[c] = new ButterworthFilter(200f, sampleRate, ButterworthFilter.PassType.Highpass);
        }

        float[] instantEnergyLow = new float[totalBlocks];
        float[] instantEnergyHigh = new float[totalBlocks];

        for (int i = 0; i < totalBlocks; i++)
        {
            float sumLow = 0f;
            float sumHigh = 0f;
            int startSample = i * blockSize;

            for (int j = 0; j < blockSize; j++)
            {
                int sampleIdx = startSample + j;
                if (sampleIdx >= rawSamples.Length) break;

                float input = rawSamples[sampleIdx];
                int channelIdx = j % channels;

                float lowSample = lowFilters[channelIdx].Process(input);
                float highSample = highFilters[channelIdx].Process(input);

                sumLow += lowSample * lowSample;
                sumHigh += highSample * highSample;
            }

            instantEnergyLow[i] = Mathf.Sqrt(sumLow / blockSize);
            instantEnergyHigh[i] = Mathf.Sqrt(sumHigh / blockSize);
        }

        // 一阶差分
        float[] diff = new float[totalBlocks];
        for (int i = 1; i < totalBlocks; i++)
        {
            float diffLow = Mathf.Max(instantEnergyLow[i] - instantEnergyLow[i - 1], 0f);
            float diffHigh = Mathf.Max(instantEnergyHigh[i] - instantEnergyHigh[i - 1], 0f);
            
            diff[i] = diffLow + diffHigh * 0.85f;
        }

        float[] avgDiff = new float[totalBlocks];
        int windowRadius = 20;
        for (int i = 0; i < totalBlocks; i++)
        {
            float sum = 0f;
            int startWin = Mathf.Max(0, i - windowRadius);
            int endWin = Mathf.Min(totalBlocks - 1, i + windowRadius);
            int localCount = endWin - startWin + 1;

            for (int j = startWin; j <= endWin; j++)
            {
                sum += diff[j];
            }
            avgDiff[i] = sum / localCount;
        }

        float lastTriggerTime = -refractoryPeriod;

        for (int i = 1; i < totalBlocks - 1; i++)
        {
            float val = diff[i];

            float totalRawEnergy = instantEnergyLow[i] + instantEnergyHigh[i];
            if (totalRawEnergy < minimumAbsoluteEnergy) continue;

            float referenceAvg = Mathf.Max(avgDiff[i], 0.001f);

            // 基于局部最大值(Local Peaks)判定起音
            if (val > diff[i - 1] && val > diff[i + 1] && val > referenceAvg * thresholdMultiplier)
            {
                float timeInSeconds = (float)(i * blockSize) / (sampleRate * channels);
                if (timeInSeconds - lastTriggerTime >= refractoryPeriod)
                {
                    detectedOnsets.Add(timeInSeconds);
                    lastTriggerTime = timeInSeconds;
                }
            }
        }

        return detectedOnsets;
    }

    private AudioClip TrimSilence(AudioClip original, float threshold, float fadeTime)
    {
        int channels = original.channels;
        int hz = original.frequency;
        float[] samples = new float[original.samples * channels];
        original.GetData(samples, 0);

        int firstSampleIndex = 0;
        for (int i = 0; i < samples.Length; i++)
        {
            if (Mathf.Abs(samples[i]) > threshold)
            {
                firstSampleIndex = i - (i % channels);
                break;
            }
        }

        int lastSampleIndex = samples.Length - 1;
        for (int i = samples.Length - 1; i >= 0; i--)
        {
            if (Mathf.Abs(samples[i]) > threshold)
            {
                lastSampleIndex = i + (channels - 1 - (i % channels));
                break;
            }
        }

        int trimmedLength = lastSampleIndex - firstSampleIndex + 1;
        if (trimmedLength <= 0) return original;

        float[] trimmedSamples = new float[trimmedLength];
        Array.Copy(samples, firstSampleIndex, trimmedSamples, 0, trimmedLength);

        int fadeSamples = Mathf.Min(Mathf.RoundToInt(hz * fadeTime) * channels, trimmedLength / 10);
        for (int i = 0; i < fadeSamples; i += channels)
        {
            float t = (float)i / fadeSamples;
            for (int c = 0; c < channels; c++)
            {
                trimmedSamples[i + c] *= t;
                trimmedSamples[trimmedSamples.Length - 1 - i - c] *= t;
            }
        }

        AudioClip newClip = AudioClip.Create(original.name + "_trimmed", trimmedLength / channels, channels, hz, false);
        newClip.SetData(trimmedSamples, 0);
        return newClip;
    }

    #endregion

    #region 内部类：极高精度全频段小数 BPM 搜寻辅助器

    private static class UniBpmAnalyzerHelper
    {
        public struct BpmMatchData
        {
            public int bpm;
            public float match;
        }

        private const int MIN_BPM = 60;
        private const int MAX_BPM = 240; 
        private const int BASE_FREQUENCY = 44100;
        private const int BASE_CHANNELS = 2;
        private const int BASE_SPLIT_SAMPLE_SIZE = 2205;

        private static BpmMatchData[] bpmMatchDatas = new BpmMatchData[MAX_BPM - MIN_BPM + 1];

        public static float AnalyzeBpm(AudioClip clip, float filterCutoff)
        {
            for (int i = 0; i < bpmMatchDatas.Length; i++)
            {
                bpmMatchDatas[i].match = 0f;
            }

            int frequency = clip.frequency;
            int channels = clip.channels;
            int splitFrameSize = Mathf.FloorToInt(((float)frequency / (float)BASE_FREQUENCY) * ((float)channels / (float)BASE_CHANNELS) * (float)BASE_SPLIT_SAMPLE_SIZE);

            var allSamples = new float[clip.samples * channels];
            clip.GetData(allSamples, 0);

            float dt = 1f / frequency;
            float RC = 1f / (2f * Mathf.PI * filterCutoff);
            float alpha = dt / (RC + dt);
            for (int c = 0; c < channels; c++)
            {
                float prev = 0f;
                for (int i = c; i < allSamples.Length; i += channels)
                {
                    allSamples[i] = prev + alpha * (allSamples[i] - prev);
                    prev = allSamples[i];
                }
            }

            var volumeArr = CreateVolumeArray(allSamples, splitFrameSize);

            int baseIntBpm = SearchIntegerBpm(volumeArr, frequency, splitFrameSize);

            float finalFloatBpm = SearchDecimalBpm(volumeArr, frequency, splitFrameSize, baseIntBpm);

            if (finalFloatBpm > 140f)
            {
                float halfBpm = finalFloatBpm / 2f;
                int halfIntBpm = Mathf.RoundToInt(halfBpm);
                BpmMatchData halfBpmData = Array.Find(bpmMatchDatas, x => x.bpm == halfIntBpm);
                BpmMatchData maxBpmData = Array.Find(bpmMatchDatas, x => x.bpm == Mathf.RoundToInt(finalFloatBpm));

                if (halfBpmData.bpm >= MIN_BPM && halfBpmData.match >= maxBpmData.match * 0.75f)
                {
                    finalFloatBpm = halfBpm;
                }
            }

            return finalFloatBpm;
        }

        private static float[] CreateVolumeArray(float[] allSamples, int splitFrameSize)
        {
            var volumeArr = new float[Mathf.CeilToInt((float)allSamples.Length / (float)splitFrameSize)];
            int powerIndex = 0;

            for (int sampleIndex = 0; sampleIndex < allSamples.Length; sampleIndex += splitFrameSize)
            {
                float sum = 0f;
                for (int frameIndex = sampleIndex; frameIndex < sampleIndex + splitFrameSize; frameIndex++)
                {
                    if (allSamples.Length <= frameIndex) break;
                    float absValue = Mathf.Abs(allSamples[frameIndex]);
                    if (absValue > 1f) continue;
                    sum += (absValue * absValue);
                }
                volumeArr[powerIndex] = Mathf.Sqrt(sum / splitFrameSize);
                powerIndex++;
            }

            float maxVolume = volumeArr.Max();
            if (maxVolume > 0)
            {
                for (int i = 0; i < volumeArr.Length; i++)
                {
                    volumeArr[i] = volumeArr[i] / maxVolume;
                }
            }
            return volumeArr;
        }

        private static int SearchIntegerBpm(float[] volumeArr, int frequency, int splitFrameSize)
        {
            var diffList = new List<float>();
            for (int i = 1; i < volumeArr.Length; i++)
            {
                diffList.Add(Mathf.Max(volumeArr[i] - volumeArr[i - 1], 0f));
            }

            int index = 0;
            float splitFrequency = (float)frequency / (float)splitFrameSize;
            float centerBpm = 120f;
            float sigma = 50f;

            for (int bpm = MIN_BPM; bpm <= MAX_BPM; bpm++)
            {
                float sinMatch = 0f;
                float cosMatch = 0f;
                float bps = (float)bpm / 60f;

                if (diffList.Count > 0)
                {
                    for (int i = 0; i < diffList.Count; i++)
                    {
                        sinMatch += (diffList[i] * Mathf.Cos(i * 2f * Mathf.PI * bps / splitFrequency));
                        cosMatch += (diffList[i] * Mathf.Sin(i * 2f * Mathf.PI * bps / splitFrequency));
                    }
                    sinMatch *= (1f / (float)diffList.Count);
                    cosMatch *= (1f / (float)diffList.Count);
                }

                float rawMatch = Mathf.Sqrt((sinMatch * sinMatch) + (cosMatch * cosMatch));
                float weight = Mathf.Exp(-Mathf.Pow(bpm - centerBpm, 2) / (2f * sigma * sigma));

                bpmMatchDatas[index].bpm = bpm;
                bpmMatchDatas[index].match = rawMatch * weight;
                index++;
            }

            int matchIndex = Array.FindIndex(bpmMatchDatas, x => x.match == bpmMatchDatas.Max(y => y.match));
            return bpmMatchDatas[matchIndex].bpm;
        }

        private static float SearchDecimalBpm(float[] volumeArr, int frequency, int splitFrameSize, int bestIntBpm)
        {
            var diffList = new List<float>();
            for (int i = 1; i < volumeArr.Length; i++)
            {
                diffList.Add(Mathf.Max(volumeArr[i] - volumeArr[i - 1], 0f));
            }

            float splitFrequency = (float)frequency / (float)splitFrameSize;
            float centerBpm = 120f;
            float sigma = 50f;

            float bestFloatBpm = bestIntBpm;
            float maxMatchScore = -1f;

            float startBpm = Mathf.Max(MIN_BPM, bestIntBpm - 0.99f);
            float endBpm = Mathf.Min(MAX_BPM, bestIntBpm + 0.99f);

            for (float bpm = startBpm; bpm <= endBpm; bpm += 0.01f)
            {
                float sinMatch = 0f;
                float cosMatch = 0f;
                float bps = bpm / 60f;

                if (diffList.Count > 0)
                {
                    for (int i = 0; i < diffList.Count; i++)
                    {
                        sinMatch += (diffList[i] * Mathf.Cos(i * 2f * Mathf.PI * bps / splitFrequency));
                        cosMatch += (diffList[i] * Mathf.Sin(i * 2f * Mathf.PI * bps / splitFrequency));
                    }
                    sinMatch *= (1f / (float)diffList.Count);
                    cosMatch *= (1f / (float)diffList.Count);
                }

                float rawMatch = Mathf.Sqrt((sinMatch * sinMatch) + (cosMatch * cosMatch));
                float weight = Mathf.Exp(-Mathf.Pow(bpm - centerBpm, 2) / (2f * sigma * sigma));
                float score = rawMatch * weight;

                if (score > maxMatchScore)
                {
                    maxMatchScore = score;
                    bestFloatBpm = bpm;
                }
            }

            return bestFloatBpm;
        }
    }

    #endregion
}
```

---

## 七、 实践与参数调试指南

在音频处理中，算法是骨架，而参数微调是肌肉。为了能在各种风格中稳定运行，推荐在 Unity 的 Inspector 调试中注意以下事项：

### 1. Unity 面板序列化覆盖陷阱（非常重要）
Unity 会将脚本挂载时的 Inspector 面板数值保存在 Scene 或 Prefab 的 YAML 文件中。即使你在代码中将默认值 `minimumAbsoluteEnergy = 0.005f` 写入，**Inspector 里的旧值（例如 0.015）依然会无情地覆盖代码默认值**。
* **现象**：高能电音的吻合度依然卡在 10% 左右。
* **解决**：在 Unity 面板中，手动点击脚本右上角的 “Reset” 按钮，或者手动将 `Minimum Absolute Energy` 修改为 `0.005` 甚至更低（如 `0.002`）。

### 2. 核心参数的权衡（Trade-offs）
* **`Alignment Tolerance`（网格容差）**：如果你需要像 OSU 那样绝对稳定、没有时间抖动的刚性谱面，请将该参数收窄为 `0.03`（30ms）。同时将 `Conflict Resolution` 选为 `UseBPMGrid`。
* **`Threshold Multiplier`（差分触发敏感度）**：如果一首民谣吉他扫弦没有被抓到重音，请将此倍数由 `1.35` 调低至 `1.2` 或 `1.25`；如果是电音鼓点太杂，请调高至 `1.45`。
* **`LPF Cutoff`**：对于普通电子和流行乐，150Hz 滤波足以锁死底鼓。而对于某些重音主要集中在 300Hz 的小清新型非插电民谣（Acoustic），可以在面板中适度提高滤波上限（如 `250f`）。

