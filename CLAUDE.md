# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Background
很多企业客户使用SAN存储，并且基于SAN存储有成熟的DR解决方案，所以很多客户在采用容器技术后，依然希望基于传统的存储disaster recovery 方案实现业务的连续性。 因此有需求通过传统的SAN存储时间metro/regional DR技术实现数据的复制。

## Project Overview

**K8SDRwithStorageDR** — 依赖于存储的DR技术，实现应用的DR。

## Technology Stack

- **Target platform**: Kubernetes
- **Automation**  Ansible, Ansible Automaiton Platform 2.7
- **Lauguage**  Ansible Playbook, Shell Sript and Python

## Development Commands

_To be filled in as the project structure is established. Common patterns to follow once tooling is set up:_

- Install dependencies: `pip install -r requirements.txt` or per the chosen package manager (pip / uv / poetry / pdm)
- Run tests: `pytest` (or `python -m pytest`)
- Lint: `ruff check .` (`.ruff_cache/` is already gitignored)
- Type check: `mypy .` (`.mypy_cache/` is already gitignored)
