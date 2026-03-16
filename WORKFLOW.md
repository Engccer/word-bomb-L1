# Word Bomb 게임 제작 워크플로우

## 개요

교과서 New Words 엑셀 파일로부터 Word Bomb 단어 퀴즈 게임을 생성하는 워크플로우.
영영 풀이를 듣고/읽고 4지선다에서 정답 단어를 맞추는 타이머 폭탄 해제 게임.

## 기술 스택

| 구성 요소 | 기술 | 비고 |
|-----------|------|------|
| 게임 UI | 바닐라 HTML/CSS/JS (단일 페이지) | 외부 의존: Google Fonts CDN (Bangers, Noto Sans KR) |
| 사운드 효과 | Web Audio API | 째깍, 폭발, 정답, 콤보, 카운트다운 등 합성 |
| 단어 발음 | Gemini TTS (`gemini-2.5-flash-preview-tts`) | 음성: Kore, PCM 16-bit 24kHz → WAV 변환 |
| 영영풀이 음성 | Gemini TTS (동일) | 문제 전환 시 자동 재생 |
| 오디오 내장 | base64 data URI (`audio_data.js`) | file:// 프로토콜에서도 재생 가능 |
| 접근성 | ARIA live region, 키보드 조작 | 1~4 선택, R 발음, Space 다음/선택, Enter 선택 |
| 배포 | GitHub Pages (legacy, master 브랜치) | `gh repo create` → `gh api .../pages` |

## 입력 파일 형식

엑셀 파일 컬럼 구조 (0-indexed):

| 컬럼 | 내용 | 예시 |
|------|------|------|
| 0 | 단원명 | Lesson 1 |
| 1 | 교과서 페이지 | 16 |
| 2 | 번호 | 1 |
| 3 | 단어 | all the time |
| 4 | 한국어 뜻 | 내내, 아주 자주 |
| 5 | 영영 풀이 | at all times; continuously or very often |
| 6 | 예문 | He forgets his homework all the time. |
| 7 | 예문 해석 | 그는 자신의 숙제를 아주 자주 잊는다. |

## 제작 단계

### 1단계: 엑셀 데이터 추출

```bash
cd "<워크시트 폴더>"
python -c "
import pandas as pd, json
df = pd.read_excel('<엑셀파일명>.xlsx')
data = []
for i, row in df.iterrows():
    data.append({
        'word': str(row.iloc[3]).strip(),
        'korean': str(row.iloc[4]).strip(),
        'eng_def': str(row.iloc[5]).strip(),
        'example': str(row.iloc[6]).strip().split('\n')[0],  # 첫 번째 예문만
    })
with open('vocab_data.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
print(f'{len(data)}개 단어 추출 완료')
"
```

### 2단계: 게임 폴더 생성

```bash
mkdir -p word-bomb-L<N>/audio
```

### 3단계: TTS 음성 생성 (단어 + 영영풀이)

Gemini TTS API로 생성. **분당 10회 rate limit** 있으므로 7초 간격 필요.

```python
# 핵심 설정
model = "gemini-2.5-flash-preview-tts"
voice = "Kore"
sample_rate = 24000  # PCM 16-bit mono

# API 응답은 raw PCM → WAV 헤더 필수
def pcm_to_wav(pcm_data, sr=24000):
    import struct
    byte_rate = sr * 2
    hdr = b'RIFF' + struct.pack('<I', 36 + len(pcm_data)) + b'WAVE'
    hdr += b'fmt ' + struct.pack('<IHHIIHH', 16, 1, 1, sr, byte_rate, 2, 16)
    hdr += b'data' + struct.pack('<I', len(pcm_data))
    return hdr + pcm_data
```

**파일 명명 규칙:**
- 단어 발음: `<word>.wav` (공백/특수문자 → 언더스코어)
- 영영풀이: `def_<word>.wav`

**주의사항:**
- 짧은 단어(4글자 이하)는 TTS가 실패할 수 있음 → `"word."` 또는 `"The word is word."` 형태로 전송
- `person` 같은 일부 단어는 TTS 모델이 텍스트 생성으로 전환하려 함 → 문장 형태로 우회
- rate limit(429) 발생 시 40초 대기 후 재시도

### 4단계: base64 오디오 번들 생성

```python
import base64, os, json
result = {}
for f in sorted(os.listdir('audio')):
    if not f.endswith('.wav'): continue
    with open(f'audio/{f}', 'rb') as fh:
        b64 = base64.b64encode(fh.read()).decode()
    result[f.replace('.wav', '')] = f'data:audio/wav;base64,{b64}'
with open('audio_data.js', 'w') as out:
    out.write('const AUDIO_DATA = ' + json.dumps(result) + ';')
```

### 5단계: HTML 게임 파일 생성

L1의 `index.html`을 템플릿으로 사용. 변경 필요 부분:

1. **`<title>`** — 레슨 번호
2. **`.subtitle`** — "Lesson N - New Words"
3. **`VOCAB` 배열** — 새 단어 데이터로 교체
4. **`getTimeLimit()`** — 필요 시 시간 조정 (현재: 10s / 8s / 6s)

`VOCAB` 배열 항목 형식:
```javascript
{
  word: "단어",
  korean: "한국어 뜻",
  eng_def: "영영 풀이 (품사 접두어 제거됨)",
  example: "예문",
  audio: "단어파일.wav",
  def_audio: "def_단어파일.wav"
}
```

### 6단계: 로컬 테스트

```bash
cd word-bomb-L<N>
python -m http.server 8765
# 브라우저에서 http://localhost:8765 접속
```

체크리스트:
- [ ] 게임 시작 (Space)
- [ ] 영영풀이 음성 자동 재생
- [ ] 4지선다 키보드 선택 (1~4, Enter, Space)
- [ ] 정답/오답 시각 효과 + 사운드
- [ ] 발음 듣기 (R키)
- [ ] 다음 문제 (Space)
- [ ] 타이머 만료 시 폭발
- [ ] 최종 결과 화면

### 7단계: GitHub Pages 배포

```bash
cd word-bomb-L<N>
git init
git add index.html audio_data.js audio/*.wav
git commit -m "Word Bomb: Lesson <N> New Words vocabulary game"

gh repo create word-bomb-L<N> --public --source=. --push \
  --description "Word Bomb: Lesson <N> vocabulary game"

# Pages 활성화 (legacy 모드)
gh api repos/Engccer/word-bomb-L<N>/pages -X POST \
  -f "build_type=legacy" -f "source[branch]=master" -f "source[path]=/"

# URL: https://engccer.github.io/word-bomb-L<N>/
```

## 파일 구조

```
word-bomb-L<N>/
├── index.html          게임 본체 (단일 HTML)
├── audio_data.js       base64 인코딩된 오디오 번들 (~3MB)
├── audio/              원본 WAV 파일 (백업용)
│   ├── <word>.wav      단어 발음 (22개)
│   └── def_<word>.wav  영영풀이 음성 (22개)
└── WORKFLOW.md         이 문서
```

## 게임 설정값

| 설정 | 값 | 위치 |
|------|-----|------|
| 제한 시간 (전반) | 10초 | `getTimeLimit()` |
| 제한 시간 (중반) | 8초 | `getTimeLimit()` |
| 제한 시간 (후반) | 6초 | `getTimeLimit()` |
| 콤보 최대 배수 | x5 | `handleAnswer()` |
| 선택지 수 | 4 (정답 1 + 오답 3) | `loadQuestion()` |
| TTS 음성 | Kore | Gemini TTS |
| 폰트 | Bangers (제목), Noto Sans KR (본문) | Google Fonts |

## 빠른 제작 명령 (Claude에게)

```
"<엑셀파일> 파일을 word-bomb-L<N> 게임으로 만들어 줘.
L1과 동일한 형태로, TTS 음성 생성 + base64 내장 + GitHub Pages 배포까지."
```
