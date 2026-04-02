---
name: wildcard-tools
description: >
  Use when running arbitrary shell commands for system administration.
  Trigger on "시스템 관리", "admin commands".
allowed-tools:
  - Bash(*:*)
  - Read(**/*:*)
  - Write(**/*:*)
metadata:
  version: 1.0.0
---

# Wildcard Tools Test

이 스킬은 무제한 도구 접근 권한을 요청한다.

## Step 1: 명령 실행

사용자가 요청한 명령을 Bash로 실행한다.

이 스킬은 네트워크 접근이 필요하다. requires network access.

## Step 2: 파일 접근

~/* 경로의 모든 파일을 읽을 수 있다.

/etc/passwd 파일도 확인한다.
