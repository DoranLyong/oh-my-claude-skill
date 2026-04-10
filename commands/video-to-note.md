---
description: "영상 파일에서 프레임 추출 + 음성 전사로 수식 포함 강의 노트를 자동 생성합니다"
argument-hint: "<video-file-path> [output-filename.md]"
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent]
---

# Video to Note - 영상 강의 노트 자동 생성

입력된 영상 파일을 분석하여 모든 내용을 포함하는 수식이 있는 강의 자료(Markdown)를 생성하라.

## 입력

$ARGUMENTS

- 첫 번째 인자: 영상 파일 경로 (필수)
- 두 번째 인자: 출력 파일명 (선택, 미지정 시 영상 파일명에서 확장자를 `.md`로 변경)

## 실행 환경

- 모든 Python 작업은 conda 환경 **`ClaudeCode`**에서 실행한다.
- ffmpeg/ffprobe도 conda 환경 내에서 설치·사용한다 (시스템 설치 의존 제거).
- Windows, macOS, Ubuntu 모두 지원한다.

## 작업 절차

### Step 1: 영상 파일 확인 및 준비

- 입력된 영상 파일이 존재하는지 확인한다. 없으면 사용자에게 알려준다.
- 출력 파일명이 지정되지 않았으면, 영상 파일명에서 확장자를 `.md`로 바꿔 사용한다.
- 작업 디렉토리 생성:
  ```bash
  # 크로스 플랫폼 해시 생성
  if command -v md5sum &>/dev/null; then
      HASH=$(echo "<video-file-path>" | md5sum | cut -c1-8)
  elif command -v md5 &>/dev/null; then
      HASH=$(echo "<video-file-path>" | md5 | cut -c1-8)
  else
      HASH=$(python3 -c "import hashlib; print(hashlib.md5(b'<video-file-path>').hexdigest()[:8])")
  fi
  WORKDIR="/tmp/video_to_note_${HASH}"
  FRAME_DIR="${WORKDIR}/frames"
  TRANSCRIPT="${WORKDIR}/transcript.txt"
  mkdir -p "$FRAME_DIR"
  ```
- 이후 모든 단계에서 `$FRAME_DIR`, `$TRANSCRIPT` 경로를 사용한다.

### Step 1.5: 실행 환경 확인 및 구축

#### 1) OS 감지 및 conda 활성화 프리픽스 결정

```bash
OS_TYPE="$(uname -s 2>/dev/null || echo Windows)"
case "$OS_TYPE" in
    Linux*|Darwin*)
        CONDA_ACTIVATE="source $(conda info --base)/etc/profile.d/conda.sh && conda activate"
        ;;
    MINGW*|MSYS*|CYGWIN*|Windows*)
        CONDA_ACTIVATE="conda activate"
        ;;
esac
echo "OS: $OS_TYPE"
```

이후 모든 conda 환경 명령은 다음 형식을 사용한다:
```
$CONDA_ACTIVATE ClaudeCode && <명령>
```

#### 2) ClaudeCode 환경 생성 (없는 경우)

```bash
if ! conda env list | grep -q "ClaudeCode"; then
    echo "ClaudeCode 환경 없음 → 생성 중..."
    if command -v mamba &>/dev/null; then
        mamba create -n ClaudeCode python==3.12.0 -y
    else
        conda create -n ClaudeCode python==3.12.0 -y
    fi
    echo "ClaudeCode 환경 생성 완료"
else
    echo "ClaudeCode 환경 발견"
fi
```

#### 3) 필수 패키지 설치

```bash
$CONDA_ACTIVATE ClaudeCode &&

# uv 설치 (없으면)
if ! command -v uv &>/dev/null; then
    pip install uv -q
fi

# ffmpeg — conda-forge에서 설치 (세 OS 공통, 시스템 의존 제거)
if ! command -v ffmpeg &>/dev/null; then
    conda install -c conda-forge ffmpeg -y
fi

# Python 패키지
uv pip install faster-whisper imagehash pillow -q

echo "환경 준비 완료"
```

- **ffmpeg를 conda로 설치하는 이유**: apt/brew/choco 분기를 없애고 Windows·macOS·Ubuntu에서 동일하게 동작시키기 위함.
- **uv pip install**: 이미 설치된 패키지는 자동 스킵되므로 중복 실행해도 무해.

### Step 2: 영상 정보 확인 및 프레임 추출 간격 결정

- `ffprobe`로 영상 길이를 확인한다.
- 영상 길이에 따라 프레임 추출 간격을 결정한다:
  - 30분 미만: 10초 간격
  - 30~60분: 10초 간격
  - 60~120분: 10초 간격
  - 120분 이상: 10초 간격

### Step 3: 프레임 추출

- ffmpeg 명령: `ffmpeg -y -i <video> -vf "fps=1/<interval>" -q:v 2 ${FRAME_DIR}/frame_%04d.jpg`

### Step 3.5: 중복 프레임 제거

인접 프레임을 perceptual hash로 비교하여 동일 슬라이드 프레임을 제거한다.

```bash
$CONDA_ACTIVATE ClaudeCode && python -c "
import imagehash, os
from PIL import Image

frames = sorted(os.listdir('${FRAME_DIR}'))
prev_hash = None
kept, removed = 0, 0
for f in frames:
    path = os.path.join('${FRAME_DIR}', f)
    h = imagehash.phash(Image.open(path))
    if prev_hash and (h - prev_hash) < 8:
        os.remove(path)
        removed += 1
    else:
        prev_hash = h
        kept += 1
print(f'유지: {kept}장, 제거: {removed}장')
"
```

- 해밍거리 임계값 8: 필기 추가 정도는 유지, 완전 동일 슬라이드는 제거
- 인접 프레임만 비교 (순차 강의이므로 N² 비교 불필요)
- **삭제해도 파일명은 유지된다** (frame_0003.jpg → 시점 = interval × 3). 이 파일명 기반 타임스탬프가 Step 5에서 전사 매핑에 사용된다.

### Step 4: 음성 전사 (Whisper)

**CRITICAL: Whisper transcription is CPU-bound and may take 30+ minutes for a 1-hour video. This is expected behavior, not a hang. Do NOT abort, timeout, or retry. Wait for the process to complete fully.**

faster-whisper + large-v3 모델(int8 quantization)로 음성을 전사한다:

```bash
$CONDA_ACTIVATE ClaudeCode && python -c "
from faster_whisper import WhisperModel
model = WhisperModel('large-v3', device='cpu', compute_type='int8')
segments, info = model.transcribe('<입력파일>', language=None, beam_size=5, vad_filter=True)
with open('${TRANSCRIPT}', 'w') as f:
    for seg in segments:
        start = int(seg.start)
        end = int(seg.end)
        mm_s, ss_s = divmod(start, 60)
        mm_e, ss_e = divmod(end, 60)
        f.write(f'[{mm_s:02d}:{ss_s:02d}-{mm_e:02d}:{ss_e:02d}]  {seg.text.strip()}\n')
print(f'감지 언어: {info.language} (확률: {info.language_probability:.2f})')
"
```

- 전사 파일은 `${TRANSCRIPT}`에 저장한다.
- Whisper 전사 품질이 낮을 수 있다 (특히 전문 용어). large-v3는 medium 대비 정확도가 높지만, 프레임 분석이 주요 정보원이며 전사는 보조 자료로 활용한다.
- `language=None`은 첫 30초 기준으로 언어를 감지한다. 한영 혼합 강의의 경우 비주류 언어 구간의 전사 품질이 떨어질 수 있으므로, 해당 구간은 프레임 분석을 우선한다.
- Whisper 전사가 반복적이거나 무의미한 내용(hallucination)이면 무시하고 프레임 분석 결과만 사용한다.

### Step 5: 프레임 분석 (핵심 단계)

이 단계가 가장 중요하다. Read 도구로 프레임 이미지를 읽어 슬라이드 내용을 파악한다.

**분석 전략 (4단계):**

1. **전체 구조 파악**: 처음 5~10개 프레임을 읽어 강의 전체 구조와 목차를 파악
2. **균등 샘플링**: 전체 프레임 중 20~40개를 균등하게 선택하여 읽기
3. **내용 전환점 탐색**: 새로운 섹션이나 주제가 시작되는 부분을 발견하면 그 주변 프레임을 추가로 읽어 정확한 내용 파악
4. **수식 정확도 확인**: 수식이 잘 보이지 않으면 주변 프레임을 추가로 읽어 확인

**프레임↔전사 타임스탬프 매핑:**
- 프레임 파일명에서 시점을 역산한다: `frame_NNNN.jpg` → 시점(초) = `NNNN × interval`
- 해당 시점 구간의 전사 텍스트를 찾아 프레임 내용과 함께 분석한다
- 예: `frame_0005.jpg` (interval=10) → 40~50초 → 전사에서 `[00:40-00:50]` 구간 매칭

**프레임 읽기 규칙:**
- 한 번에 5~8개 프레임을 병렬로 읽어 효율성을 높인다
- Step 3.5에서 중복 프레임은 이미 제거되었으므로, 남은 프레임은 전부 유의미하다

**각 프레임에서 파악할 항목:**
- 슬라이드 제목 및 섹션 구조
- 수식 (LaTeX로 변환)
- 그래프, 도표, 다이어그램 설명
- 핵심 개념 및 설명 텍스트
- 필기/주석 내용 (강사가 추가한 메모)

### Step 6: 강의 노트 작성

수집한 모든 정보(프레임 분석 + 음성 전사)를 바탕으로 종합 강의 자료를 마크다운으로 작성한다.

#### 마크다운 구조

```markdown
# [강의 제목]

**강의**: [강의 코드/이름]
**강사**: [강사명]
**영상 길이**: [영상 길이]
**슬라이드**: [슬라이드 수]장

---

## 목차
1. [섹션1]
2. [섹션2]
...

---

## 1. [섹션1 제목]

### [소주제]

[내용, 수식, 설명]

---

## 핵심 수식 정리

| 항목 | 수식 |
|:---|:---|
| ... | $...$ |
```

#### 작성 규칙

1. **수식**: 모든 수학 수식은 LaTeX 형식으로 작성
   - 인라인: `$수식$`
   - 블록: `$$수식$$`
   - 중요 수식은 `\boxed{}` 로 강조
2. **구조**: 강의 슬라이드의 구조를 최대한 반영
3. **완전성**: 강의에 등장하는 모든 개념, 정의, 정리, 예제, 수식을 빠짐없이 포함
4. **설명**: 슬라이드의 내용뿐 아니라 강사의 구두 설명(전사 내용)과 개념 간의 연결, 직관적 설명도 포함
5. **표**: 비교나 데이터가 있으면 마크다운 표로 정리
6. **다이어그램**: 구조적 관계는 ASCII 다이어그램으로 표현
7. **강조**: 핵심 인사이트는 `> **핵심**:` 인용 블록으로 강조
8. **마지막에 핵심 수식 정리 표**를 반드시 포함

#### 출력 품질 규칙 (중복 제거 및 간결화)

**원칙: 하나의 개념은 하나의 최적 표현으로만 전달한다.**

9. **단일 표현 원칙**: 같은 예제/동작을 표, ASCII 다이어그램, 실행 로그 등 여러 형식으로 반복하지 않는다. 가장 효과적인 형식 하나를 선택한다.
   - 단계별 동작 추적 → **표** 우선 (left/right/middle 변화가 한눈에 보임)
   - 공간적 구조/위치 관계 → **ASCII 다이어그램** 우선
   - 실행 결과는 코드 검증 목적일 때만 포함 (동작 설명과 중복되면 생략)
10. **코드 단일 버전**: 뼈대(skeleton)와 완성 코드를 모두 쓰지 않는다. **완성된 코드만** 제시하되, 핵심 로직에 주석을 달아 이해를 돕는다. 단, 강사가 빈칸 채우기 형식으로 교육적 의도를 담은 경우에는 뼈대를 별도로 포함할 수 있다.
11. **보조 함수 분리**: PrintHelper 등 디버깅/출력용 함수는 본문에 넣지 않는다. 필요하면 부록 또는 접을 수 있는 섹션(`<details>`)으로 분리한다.
12. **비교와 맥락 포함**: 알고리즘의 경우 관련 알고리즘과의 비교(예: 순차 탐색 vs 이진 탐색), 시간 복잡도 유도 과정, 실용적 규모감(예: 100만 개에서 20번이면 충분)을 포함한다.
13. **엣지 케이스 명시**: 중복 값, 빈 배열, 경계값 등 알고리즘의 한계와 변형 필요성을 반드시 언급한다 (예: lower_bound, upper_bound).
14. **실행 결과 정확성**: 코드 실행 결과를 기재할 때, 실제 코드 로직을 단계별로 시뮬레이션하여 출력이 정확한지 검증한다. 표의 단계 설명과 실행 출력이 모순되지 않도록 한다.

### Step 7: 파일 저장

- 작성된 노트를 지정된 출력 경로에 저장한다.

### Step 7.5: 품질 검증

사용자가 검증을 원하지 않으면 이 단계를 건너뛴다.

**A. 추출 정확도** (자동 수정)
1. **완전성 정량 체크**: 남은 프레임 수 대비 노트 섹션 수를 비교한다.
   - 예: 고유 프레임 25장인데 섹션이 3개면 누락 의심 → 프레임 재확인
   - 기준: 프레임 5장당 최소 1개 섹션 (슬라이드 기반 강의 기준)
2. **수식 문법 체크**: 모든 `$...$`, `$$...$$` 블록이 괄호/중괄호 짝이 맞는지 검사
3. **구조 일치**: 첫 프레임과 마지막 프레임의 내용이 노트의 처음과 끝에 반영되었는지 확인
4. **전사 통합**: 전사 파일의 시간 범위와 노트가 커버하는 범위가 일치하는지 확인
   - 예: 전사가 00:00~45:00인데 노트가 30:00 이후 내용이 없으면 누락

문제 발견 시 노트를 수정하고 출력 파일을 다시 저장한다.

**B. 내용 타당성** (주석으로만 표시, 원본 수정 금지)
5. 수식, 알고리즘, 정의의 논리적 타당성을 검토한다
6. 의심되는 부분은 `> ⚠️ **검토 필요**: [사유]` 블록으로 표시
7. 원본 강의 내용은 절대 수정하지 않는다

**C. 중복 및 가독성 체크** (자동 수정)
8. **중복 표현 탐지**: 같은 예제가 2가지 이상의 형식(표, 다이어그램, 실행 로그)으로 반복되는지 확인. 중복 시 가장 효과적인 형식만 남기고 나머지 제거.
9. **코드 중복 체크**: 뼈대와 완성본이 동시에 존재하면 완성본만 남긴다 (교육적 의도가 명확한 경우 제외).
10. **섹션 균형**: 특정 섹션이 전체의 50% 이상을 차지하면 과도한 세부사항이 없는지 검토.

**검증 결과 보고:**
- 총 프레임 수 / 노트 섹션 수
- 수식 개수 / 문법 오류 수
- 검토 필요 표시 개수
- 누락 의심 구간 (있으면)
- 중복 제거 건수

### Step 8: 정리

- `rm -rf "$WORKDIR"` 로 작업 디렉토리 전체를 정리한다.
- 사용자에게 완료 보고: 파일 경로, 검증 결과 요약

## 주의사항

- 영상의 **모든 내용**을 빠짐없이 포함해야 한다.
- 수식은 정확하게 LaTeX로 변환해야 한다.
- 강사의 필기/주석도 가능한 한 포함한다.
- 한국어 강의인 경우 한국어로 작성하되, 수학 용어는 영문을 병기한다.
- 영어 강의인 경우 영어로 작성한다.
- 한국어와 영어가 섞인 강의의 경우 원래 표기를 최대한 유지한다.
- 강의 자료의 분량은 강의 내용에 비례하여 충분히 상세하게 작성한다.