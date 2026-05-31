---
layout: default
title: Profile
permalink: /profile/
---

<div class="resume">
  <div class="resume-header">
    <div>
      <h1 class="resume-name">Tianer Zhou</h1>
      <p class="resume-role">Graphics Engineer • OpenHarmony, Huawei</p>
    </div>

    <div class="resume-contact">
      <a href="mailto:ezhoureal@gmail.com">ezhoureal@gmail.com</a>
      <a href="https://www.linkedin.com/in/tianer-zhou/" target="_blank" rel="noopener">LinkedIn</a>
      <a href="https://github.com/ezhoureal" target="_blank" rel="noopener">GitHub</a>
      <span class="resume-location">Hangzhou, China</span>
      <button onclick="window.print()" class="resume-print">Print / PDF</button>
    </div>
  </div>

  <p class="resume-intro">
    I work on the open-source graphics and UI foundations of HarmonyOS at Huawei. My focus is building performant, reliable rendering systems that run on millions of devices.
  </p>

  <section class="resume-section">
    <h2 class="resume-section-title">Education</h2>
    <div class="resume-entry">
      <div class="resume-entry-head">
        <strong>University of Michigan, Ann Arbor</strong>
        <span class="resume-date">2019 — 2022</span>
      </div>
      <div class="resume-entry-sub">B.S.E. Computer Science, <em>Summa Cum Laude</em></div>
      <p>Projects: deveoped basic OS kernel (threads, virtual memory, file system), implemented Paxos and sharded workers, and created a PID-controlled Arduino drone that placed in the top 5% of the course.</p>
    </div>
  </section>

  <section class="resume-section">
    <h2 class="resume-section-title">Experience</h2>

    <div class="resume-entry">
      <div class="resume-entry-head">
        <strong>Huawei Technologies</strong>
        <span class="resume-date">2022 — Present</span>
      </div>
      <div class="resume-entry-sub">Graphics Engineer, HarmonyOS Graphics Foundation</div>
      <p>Worked on the rendering stack now shipping on 25M+ devices. Developed visual-effect shaders and automated parameter search methods that improved performance up to 10% in production shaders. Explored on-device 3D Gaussian rendering and lightweight generative models for mobile.</p>
    </div>

    <div class="resume-entry">
      <div class="resume-entry-sub">UI Framework Engineer, native UI framework</div>
      <p>Core contributor (1,200+ commits, #1 out of 1,300+) to the open-source UI engine that powers 20,000+ HarmonyOS apps. Built high-performance scroll containers that reduced jank by 60%, a production image loader, and a React Native SVG renderer 20% faster than the Android reference.</p>
    </div>
  </section>

  <section class="resume-section">
    <h2 class="resume-section-title">Selected Projects</h2>
    <ul class="resume-bullets resume-projects">
      <li><strong><a href="https://github.com/ezhoureal/myGPT" target="_blank" rel="noopener">myGPT</a></strong> — From-scratch GPT implementation (BPE tokenizer, transformer, FlashAttention, distributed KV cache, gRPO post-training).</li>
      <li><strong><a href="https://github.com/ezhoureal/deepRL" target="_blank" rel="noopener">deepRL</a></strong> — SAC, IQL, AWAC, PPO from scratch. Strong results on Ant Maze (Berkeley CS 285).</li>
      <li><strong><a href="https://github.com/ezhoureal/snake-compiler" target="_blank" rel="noopener">snake-compiler</a></strong> — Rust x86 compiler for a JavaScript-like language with closures and dynamic typing.</li>
      <li>Contributions to <strong>vLLM</strong> and <strong>NVIDIA Dynamo</strong> (distributed LLM inference). Built HarmonyOS MCP for LLM–device UI interaction.</li>
    </ul>
  </section>

  <div class="resume-footer">
    <p>Always happy to chat about rendering, on-device ML, or LLM infra.</p>
    <a href="https://github.com/ezhoureal" class="btn btn-primary" target="_blank" rel="noopener">GitHub →</a>
  </div>
</div>
