# Chrome Issue 992914 — 32-bit Port (Windows)

> **Re-upload notice / 다시 올리는 이유**
> 2019년 Exodus Intelligence가 공개한 64비트 익스플로잇을 제가 **32비트 Windows Chrome용으로 포팅**하면서 정리해 둔 자료입니다.
> 예전에 GitHub에 올렸다가 내렸던 저장소를, 기록 보존 차원에서 그대로 다시 공개합니다. (작성 시점: 2019~2021, 내용은 당시 그대로이며 이후 갱신 없음)
>
> This is an old project — a 32-bit Windows port of Exodus Intelligence's original 64-bit exploit for Chromium bug 992914.
> It was published here years ago, taken down, and is now **re-uploaded as-is for archival purposes**. No updates since 2021.

---

## 개요 / Overview

Chromium V8의 **sealed/frozen elements kind type confusion** (Chromium bug **992914**) 취약점 익스플로잇입니다.
원본은 Exodus Intelligence의 "patch-gapping" 연구에서 공개된 **64비트** 버전이고, 이 저장소는 이를 **32비트 Chrome (Windows)** 환경에 맞게 변환한 것입니다.

The bug is a type confusion in V8's handling of sealed/frozen element kinds. Exodus published a working 64-bit exploit as part of their patch-gapping research; this repository contains a **32-bit Windows port** of that exploit, plus the slides describing how the conversion was done.

* **Original 64-bit exploit author:** Exodus Intelligence
* **32-bit port / 32비트 변환:** this repository
* **Tested on:** Windows 10, **32-bit** Chrome 76.0.3809.100 / 76.0.3809.132

## 파일 구성 / Files

| File | Description |
| --- | --- |
| `exp_32bit.html` | 익스플로잇 진입점. `print()` / `hex()` / `hexdump()` 헬퍼를 정의하고 `chrome_992914_32bit.js`를 로드합니다. |
| `chrome_992914_32bit.js` | 브라우저용 32비트 익스플로잇 본체 (type confusion → addrof/AAR/AAW → WASM RWX → shellcode). |
| `d8_32bit_ex.js` | d8(V8 셸)에서 디버깅용으로 쓰던 변형본. 로그가 더 자세하고 일부 단계가 주석 처리되어 있습니다. |
| `How_to_convert_Chrome_Issue992914_exploit_to_32-bit_on_Windows.pptx` | 64비트 → 32비트 변환 과정을 정리한 발표 자료 (약 34MB). |

## 동작 방식 요약 / How it works

1. Sealed/frozen element kind 타입 혼동을 이용해 인접한 `float_array`의 길이/데이터 포인터를 깨뜨립니다.
2. 깨진 float 배열로 **relative OOB read/write**를 얻고, 이를 `addrof` 프리미티브로 확장합니다.
3. 32비트에서는 하나의 double(64비트) 안에 두 개의 32비트 포인터가 들어가므로, `setHighUint32()` / `float_to_low_32bit()` 같은 헬퍼로 상·하위 32비트를 나눠 다루며 **임의 주소 읽기/쓰기(AAR/AAW)** 를 구성합니다.
4. `WebAssembly.Instance` → `WasmExportedFunctionData` → instance 오브젝트를 따라가 **RWX 페이지 주소**를 얻습니다.
5. RWX 페이지에 셸코드를 쓰고, 익스포트된 WASM 함수(`wfunc()`)를 호출해 실행합니다.

기본 셸코드는 PEB 워킹으로 `WinExec`를 찾아 **`calc`** 를 실행하는 32비트 코드입니다 (`chrome_992914_32bit.js`의 `shellcode` 배열).
힙 상태에 따라 실패할 수 있어 최대 500회 반복하고, 그래도 실패하면 2초 뒤 페이지를 리로드해 재시도합니다.

## 실행 방법 / Usage

```
chrome.exe --no-sandbox
```

1. **32비트** Chrome 76.0.3809.100 또는 76.0.3809.132 를 `--no-sandbox` 로 실행합니다.
2. 이 디렉터리를 로컬 웹서버로 호스팅합니다. (예: `python -m http.server 8000`)
3. 브라우저에서 `http://localhost:8000/exp_32bit.html` 접속.
4. 성공하면 계산기가 뜹니다. 진행 로그는 DevTools 콘솔에서 확인할 수 있습니다.

d8로 디버깅할 때:

```
d8.exe --allow-natives-syntax d8_32bit_ex.js
```

## 주의 / Disclaimer

* 대상 취약점은 **2019년에 이미 패치**되었습니다. 최신 Chrome에서는 동작하지 않습니다.
* 이 코드는 **연구·교육 목적**으로만 공개합니다. 반드시 본인 소유이거나 명시적으로 허가받은 격리된 테스트 환경에서만 실행하세요.
* 셸코드가 포함되어 있으므로 실 사용 중인 머신에서 실행하지 마시고, VM에서 테스트하시기 바랍니다.
* 원본 익스플로잇의 저작권/크레딧은 Exodus Intelligence에 있습니다.

---

*Originally written 2019–2021. Re-uploaded unchanged.*
