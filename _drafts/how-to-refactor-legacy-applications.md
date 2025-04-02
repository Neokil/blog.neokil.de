---
layout: post
title: "Lets take a chunk from that"
subtitle: "..."
date:   2025-04-01 16:00:00 +0200
categories: general golang refactoring
---

Explain how refactoring legacy applications can work if you take one piece out at a time.
Example message-broker.
Things that don't change will be left as they are.
Things that change frequently will be refactored and extracted (either into package or their own repo)
