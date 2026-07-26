---
layout: default
title: Profile
permalink: /profile/
---

<div class="resume">
  <div class="resume-header">
    <div>
      <h1 class="resume-name">Tianer Zhou</h1>
      <p class="resume-role">Graphics & UI Framework Engineer • OpenHarmony, Huawei</p>
    </div>

    <div class="resume-contact">
      <a href="mailto:tianer.zhou@epfl.ch">tianer.zhou@epfl.ch</a>
      <a href="mailto:ezhoureal@gmail.com">ezhoureal@gmail.com</a>
      <a href="https://www.linkedin.com/in/tianer-zhou/" target="_blank" rel="noopener">LinkedIn</a>
      <a href="https://github.com/ezhoureal" target="_blank" rel="noopener">GitHub</a>
      <span class="resume-location">Hangzhou, China</span>
      <a href="/assets/resume.pdf" class="resume-print" target="_blank" rel="noopener">Print / PDF</a>
    </div>
  </div>

  <p class="resume-intro">
    I work on the open-source graphics and UI foundations of HarmonyOS at Huawei. My focus is building high-performance, reliable rendering systems that enrich millions of consumers.
  </p>

  <section class="resume-section">
    <h2 class="resume-section-title">Education</h2>
    <div class="resume-entry">
      <div class="resume-entry-head">
        <strong>University of Michigan, Ann Arbor</strong>
        <span class="resume-date">2019 — 2022</span>
      </div>
      <div class="resume-entry-sub">Bachelor of Science in Computer Science, <em>Summa Cum Laude</em> — GPA: 3.917/4.0</div>
      <p><strong>Selected Coursework:</strong> Machine Learning, Distributed Systems, Operating Systems, Computer Architecture, Linear Algebra, Database, Computer Security, Web Systems.</p>
      <p>Projects: developed basic OS kernel (threads, virtual memory, file system), implemented Paxos and sharded workers, and created a PID-controlled Arduino drone that placed in the top 5% of the course.</p>
    </div>
  </section>

  <section class="resume-section">
    <h2 class="resume-section-title">AI Systems &amp; Research Projects</h2>
    <ul class="resume-bullets resume-projects">
      <li><strong><a href="https://github.com/ezhoureal/HWM_PLDM/tree/up" target="_blank" rel="noopener">JEPA World Model Research</a></strong> — Reproduced hierarchical JEPA from HWM_PLDM and optimized its planning/actor loop for JEPA-style world models, achieving up to 10× planning speedup while continuing accuracy-improvement experiments.</li>
      <li><strong><a href="https://github.com/ezhoureal/le-wm/tree/predictor" target="_blank" rel="noopener">Latent Ensemble World Models</a></strong> — Trained 3-member leWM ensembles for model-based planning, and trained a separate subgoal planner to guide JEPA, increasing long-horizon task success rate from 5% to 72% in PushT.</li>
      <li><strong><a href="https://github.com/ezhoureal/myGPT" target="_blank" rel="noopener">myGPT</a></strong> — GPT training/inference stack built from scratch: BPE tokenization, Transformer blocks, RoPE, FlashAttention kernel, distributed KV cache, and gRPO post-training.</li>
      <li><strong><a href="https://github.com/ezhoureal/deepRL" target="_blank" rel="noopener">deepRL</a></strong> — SAC, IQL, AWAC, PPO implemented from scratch. IQL reached 230/300 on Ant Maze in 30k steps.</li>
      <li><strong>AI Runtime Contributions</strong> — Merged PRs to <a href="https://github.com/ai-dynamo/dynamo" target="_blank" rel="noopener">NVIDIA Dynamo</a> and <a href="https://github.com/vllm-project/vllm" target="_blank" rel="noopener">vLLM</a> (KV-router scheduler tests, out-of-bounds fix); submitted <a href="https://github.com/nearai/ironclaw" target="_blank" rel="noopener">Ironclaw</a> setup/safety fixes and issues.</li>
      <li><strong><a href="https://github.com/ezhoureal/mcp_harmony" target="_blank" rel="noopener">HarmonyOS MCP Server</a></strong> — Developed an MCP server enabling LLM agents to inspect and control HarmonyOS device UI.</li>
      <li><strong><a href="https://github.com/ezhoureal/snake-compiler" target="_blank" rel="noopener">snake-compiler</a></strong> — Rust x86 compiler for a JavaScript-like language with closures and dynamic typing.</li>
    </ul>
  </section>

  <section class="resume-section">
    <h2 class="resume-section-title">Experience</h2>

    <div class="resume-entry">
      <div class="resume-entry-head">
        <strong>Huawei Technologies</strong>
        <span class="resume-date">Aug 2022 — Aug 2026</span>
      </div>
      <div class="resume-entry-sub">Graphics Engineer, <a href="https://gitcode.com/openharmony">HarmonyOS Graphics Foundation</a> — now deployed on 70M+ devices</div>
      <ul>
        <li><strong>Rendering Pipeline &amp; Shader Optimization:</strong> Developed production 2D visual-effect shaders and an asynchronous color-extraction algorithm for adaptive glass effects; built automated parameter-search tooling to tune shader parameters and improve runtime performance up to 10%.</li>
        <li><strong>Model Fine-tuning and Deployment:</strong> Quantized and deployed a 6B-parameter diffusion model for mobile, generating 2K-resolution images on flagship mobile NPUs in 10 seconds. Curated a style-transfer dataset of 100+ entries and fine-tuned the model with LoRA on 4 V100s using FSDP.</li>
      </ul>
      <p>Exploring on-device 3D Gaussian rendering and lightweight generative models for mobile.</p>
    </div>

    <div class="resume-entry">
      <div class="resume-entry-sub">UI Framework Engineer, <a href="https://gitee.com/openharmony/arkui_ace_engine">ArkUI of HarmonyOS</a> — supporting 20,000+ apps</div>
      <p>Core contributor (1,200+ commits, ranked Top 1 among 1,300+ contributors) to the open-source UI engine. Built high-performance lazy/reusable scroll containers that reduced jank rate by 60%, designed deduplicated image caching, and created a React Native SVG renderer 20% faster than the Android reference.</p>
    </div>
  </section>

  <section class="resume-section">
    <h2 class="resume-section-title">Technical Skills</h2>
    <ul class="resume-bullets">
      <li><strong>AI Systems:</strong> Diffusion / flow matching, LLM, JEPA, Model-based Planning, RL, training / inference parallelism, KV Cache, attention kernels</li>
      <li><strong>Frameworks:</strong> PyTorch, Jax, Lightning</li>
      <li><strong>Programming Languages:</strong> C++, TypeScript, Rust, Go, Python</li>
      <li><strong>Systems:</strong> Distributed Systems, Graphics Runtime, Compiler, xv6 Kernel</li>
      <li><strong>AI Harness:</strong> Codex, Claude Code, Openclaw</li>
    </ul>
  </section>

  <div class="resume-footer">
    <p>Always happy to chat about machine learning, operating systems, and rendering.</p>
    <a href="https://github.com/ezhoureal" class="btn btn-primary" target="_blank" rel="noopener">GitHub →</a>
  </div>
</div>
