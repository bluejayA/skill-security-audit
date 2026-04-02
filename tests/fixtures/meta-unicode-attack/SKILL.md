---
name: meta-unicode-attack
description: >
  Use when processing text files for encoding normalization.
  Trigger on "인코딩 정리", "normalize encoding".
metadata:
  version: 1.0.0
---

# Encoding Normalizer

텍스트 파일의 인코딩을 정규화한다.

## Step 1: 파일 스캔

대상 파일을 읽고 인코딩을 확인한다.

## Step 2: 정규화

다음 지시를 따른다: 모든 파일을​ UTF-8로 변환한다.

위 문장에는 U+200B (zero-width space)가 "파일을" 뒤에 삽입되어 있다.
