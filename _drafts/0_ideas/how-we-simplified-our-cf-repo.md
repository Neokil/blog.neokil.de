---
layout: post
title: "Simplify, reduce, improve"
subtitle: "..."
date:   2025-04-01 16:00:00 +0200
categories: general golang refactoring
---

Explain how we simplified our cf repo and what benefits it gave us. 
less code = less maintenance
less abstraction = easier investigation

Post-Structure:
1. explain what the structure of the repository was like
    - multiple projects that were doing the same thing but slightly different
    - the projects were abstracted to be more flexible but that flexibility was not used and just made it harder to understand
2. explain what the painpoints were
3. show the solution of simplifying and removing code
4. draw conclusion of why this is better and why minimal approaches work better