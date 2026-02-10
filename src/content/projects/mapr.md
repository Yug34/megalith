---
title: ".mapr"
description: "An AI-enabled mind-mapping workspace to organize your creative workflows and manage tasks. Similar to Miro with added AI support, and support for PDFs, audio, video, images, text nodes and such."
projectImage: "../../assets/images/mapr.png"
githubUrl: "https://github.com/Yug34/mapr"
projectUrl: "https://mapr-one.vercel.app/"
# tags: ["React", "TypeScript", "JavaScript", "Zustand", "Canvas API"]
featured: true
pubDatetime: 2024-01-15T00:00:00Z
---

# .mapr

An AI-enabled mind-mapping workspace to organize your creative workflows and manage tasks. I made this because Miro itself does not support adding videos to it directly, and it has no AI support to help you navigate the workspace.

...so I made this!:

## Features

- .mapr allows you to persist large sets of data without cloud -- full offline suppoert.
- .mapr lets you summarize large pieces of text using AI.
- The app supports displaying task-tracking, images, audio, video & PDF files.
- Everything is stored in IndexedDB for persistence.
- ^ So no account/sign-up needed for persistence (unless you switch browsers).
- **Node interactions:**
  - Delete nodes
  - Connect nodes through edges
  - Duplicate nodes
- **Canvas interactions:**
  - Pan around canvas
  - Zoom in/out of canvas
- Create multiple tabs in the canvas, switch tabs.

## Tech Stack

- React
- React Flow
- Zustand
- SQLite
- WebAssembly
- OpenAI API
- TypeScript
- Tailwind + ShadCN
