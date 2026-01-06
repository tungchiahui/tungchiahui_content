---
title: 首页
path: /
---

<div class="home-header">

  <h1>🌟 欢迎来到 <b>Tung Chia-hui 的个人网站</b> 👋</h1>
  <p class="subtitle">探索、学习与创造的记录</p>

  <div class="btn-group">
    <a href="/blog" class="btn-primary">
      🚀 进入博客文章
    </a>
  </div>

</div>

---

## 🧠 关于本站

<p class="intro">
这里是我记录学习与创作的地方，主要内容包括：
</p>

<div class="cards">

  <div class="card">
    <h3>💻 编程与嵌入式开发</h3>
    <p>专注于 C/C++、Python 编程，以及基于 Linux 系统环境的开发。涉及嵌入式系统（STM32、ESP32）、FreeRTOS 与底层驱动实现。</p>
  </div>

  <div class="card">
    <h3>🤖 机器人与自动化</h3>
    <p>学习与实践 ROS1 / ROS2 平台下的运动控制、导航建图与传感器融合，同时探索 OpenCV4 在计算机视觉与环境感知中的应用。</p>
  </div>

  <div class="card">
    <h3>🖥️ 图形界面与工具开发</h3>
    <p>使用 Qt6 构建上位机与可视化调试工具。</p>
  </div>

  <div class="card">
    <h3>🌐 Web 与博客技术</h3>
    <p>本站基于 Nuxt.js 搭建，偶尔分享 Vue/ JavaScript / HTML / CSS / 前端优化的经验。</p>
  </div>

  <div class="card">
    <h3>📱 移动应用与工具开发</h3>
    <p>在 Android 平台上开发机器人上位机与辅助工具，实现控制、监控与数据可视化，提升机器人项目的效率与体验。</p>
  </div>

  <div class="card">
    <h3>✏️ 随笔与思考</h3>
    <p>记录我在学习、开发与生活中的一些想法与感悟。</p>
  </div>

</div>

---

<div class="content-page">

## 📚 最新博客
<PostList :limit="5" />

## 📖 最新 Wiki
<WikiList :limit="5" />

</div>