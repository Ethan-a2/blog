<!-- obsidian --><h1 data-heading="测试环境与配置 (Test Environment)">测试环境与配置 (Test Environment)</h1>
<ul>
<li>模型: SmolLM2-360M (超轻量级 LLM)</li>
<li>运行平台: Android 设备 (基于 Qualcomm Snapdragon 平台)</li>
</ul>
<h1 data-heading="核心数据对比分析 (Core Data Comparison)">核心数据对比分析 (Core Data Comparison)</h1>

指标维度 | 数据组 1: CPU/Host 推理 (0层 Offload) | 数据组 2: HTP 卸载 (32层 Offload) | 数据组 3: 只有2层
-- | -- | -- | --
推理框架 | llama.cpp | llama.cpp | QNN Genie
层数 | 32 | 32 | 2
Prompt Tokens | 19 (测试片段) | 19 (测试片段) | 698 (原始输入规模)
Gen Tokens | 458 | 177 | 100 (生成规模)
Prompt Eval Rate | 240.23 toks/sec | 150.10 toks/sec | 66666.66 toks/sec
Token Generation Rate | 134.60 toks/sec | 40.01 toks/sec | 800.64 toks/sec
首字延迟 (TTFT) | 79 ms | 126 ms | 11.1 ms
Total Time | 4749.14 ms | 9791.92 ms | -
Unaccounted Time % | 26.3 % | 53.4 % | -
HTP0 Compute Mem | 0 MiB | 32.26 MiB | 0 MiB
Host Memory Usage | 719 MiB | 747 MiB | -


<ul>
<li>备注：数组3只有2层，仅图一乐</li>
</ul>
<h1 data-heading="深度技术洞察 (Deep Dive Analysis)">深度技术洞察 (Deep Dive Analysis)</h1>
<h2 data-heading="A. Eval 速度断崖式下跌">A. Eval 速度断崖式下跌</h2>
<ul>
<li>第一组 (134.60 t/s)：表现正常，说明此时计算资源（CPU/Hexagon）调度顺畅。</li>
<li>第二组 (40.01 t/s)：出现了严重的性能降级。结合 <code>unaccounted time</code> 的激增，这通常意味着热节流 (Thermal Throttling) 或 系统后台任务抢占资源。由于 <code>llama.cpp</code> 在 Android/嵌入式设备上运行，长时间运行会导致 SoC 发热，系统为降温而降低 CPU/NPU 频率，从而导致 Eval 速度骤降。</li>
</ul>
<h2 data-heading="B. &#x22;Unaccounted Time&#x22; 的严重异常">B. "Unaccounted Time" 的严重异常</h2>
<ul>
<li>第一组 <code>unaccounted</code> 占比 26.3%，是相对正常的开销。</li>
<li>第二组 <code>unaccounted</code> 暴增至 53.4%。这意味着超过一半的程序运行时间没有被算入核心计算（Sampling/Eval）。
<ul>
<li>可能原因：设备内存碎片化严重、锁竞争（Lock Contention）、或者驱动程序在等待硬件响应。这也解释了为什么总时间大幅增加，但计算本身只占了一小部分。</li>
</ul>
</li>
</ul>
<h2 data-heading="C. 计算后端切换：HTP0 (Hexagon) 的差异">C. 计算后端切换：HTP0 (Hexagon) 的差异</h2>
<ul>
<li>第一组：<code>HTP0</code> 的计算缓冲大小为 <code>0.0000 MiB</code>。这意味着第一组任务完全是在 CPU 上运行，并没有利用到 Hexagon DSP/NPU 加速。</li>
<li>第二组：<code>HTP0</code> 的计算缓冲大小变为了 <code>32.2578 MiB</code>。这说明程序尝试将部分任务分发给 Hexagon 加速器。</li>
<li>反直觉结论：虽然第二组使用了 Hexagon 加速，但性能反而更慢。这说明在该设备上，Hexagon 的当前配置或驱动实现效率极低，或者在 CPU 与 NPU 之间进行数据传输（Offloading）的开销远超其带来的计算收益，从而导致了高额的 <code>unaccounted time</code>。</li>
</ul>
<h2 data-heading="D. 内存分布分析">D. 内存分布分析</h2>
<ul>
<li>Host 内存消耗：第二组比第一组多了约 28 MiB (747 - 719 MiB)。这通常与上下文的增加或重复使用导致的内存碎片有关。</li>
<li>CPU_REPACK：两组数值接近，说明模型加载状态一致。</li>
</ul>
<h1 data-heading="总结与实践建议 (Conclusion &#x26; Recommendations)">总结与实践建议 (Conclusion &#x26; Recommendations)</h1>
<ul>
<li>前两组数据对比揭示了一个典型问题：盲目启用 NPU/DSP 加速并不总能提升性能，有时反而会带来巨大的调度损耗。</li>
</ul>
<h1 data-heading="附：技术细节">附：技术细节</h1>
<h2 data-heading="llama.cpp (CPU) 关键性能指标解析">llama.cpp (CPU) 关键性能指标解析</h2>
<ul>
<li>sampling_time      18.39 ms  </li>
<li>prompt_eval        79.09 ms / 19 tok  (240.23 t/s, 4.16 ms/tok)  </li>
<li>eval               3402.79 ms / 458 runs (7.43 ms/tok, 134.60 t/s)  </li>
<li>total              4749.14 ms / 477 tok  </li>
<li>unaccounted        1248.87 ms (26.3%)  </li>
<li>graphs reused      456  </li>
<li>memory: CPU=199, REPACK=104, MODEL=199, COMPUTE=320</li>
</ul>
<h2 data-heading="llama.cpp (HTP)">llama.cpp (HTP)</h2>
<ul>
<li>load_time          482.43 ms   ← HTP graph compile 占半秒  </li>
<li>prompt_eval        126.58 ms / 19 tok (150.10 t/s, 6.66 ms/tok)  </li>
<li>eval               4423.58 ms / 177 runs (24.99 ms/tok, 40.01 t/s)  </li>
<li>total              9791.92 ms / 196 tok  </li>
<li>unaccounted        5228.44 ms (53.4%)  ← 数据回传 (rpcmem) 吞噬  </li>
<li>graphs reused      176           ← graph 缓存几乎空转  </li>
<li>memory: CPU=199, REPACK=100, HTP0-REPACK=5, COMPUTE=32</li>
</ul>
<h2 data-heading="genie-t2t-run 关键性能指标解析">genie-t2t-run 关键性能指标解析</h2>
<pre><code>{  
  "GenieDialog_create": { "duration": 268960 },  
  "GenieDialog_query": {  
    "prompt-processing-rate": 66666.66,   // 698 tok → ~10.5 ms  
    "time-to-first-token":     10819,      // 10.8 ms  
    "token-generation-rate":   801,       // 100 tok → 124.9 ms  
    "token-generation-time":   124918     // 124.9 ms  
  },  
  "GenieDialog_free": { "duration": 19399 }  
}
</code></pre>