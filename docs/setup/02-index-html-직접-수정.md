# `index.html`을 직접 고쳐야 할 때

## 이 파일의 정체

`index.html`(~7.8MB)은 사람이 짠 순수 HTML/JS 파일이 아니라 **Claude Design 캔버스에서 내보낸(export) 번들 파일**입니다. 최상위 구조는 이렇습니다.

```
<script type="__bundler/manifest">      리소스(이미지 등) base64 매니페스트
<script type="__bundler/ext_resources"> 외부 스크립트 URL 목록
<script type="__bundler/page_order">    페이지 순서 (현재 빈 배열)
<script type="__bundler/template">      실제 페이지 전체가 JS 문자열(JSON)로 이스케이프되어 들어있음
```

실제 화면 마크업·스타일·로직은 `__bundler/template` 안에 **문자열로 인코딩**되어 있고, 그 안에서 Apps Script 백엔드 코드(`doGet`/`doPost` 등)는 다시 `const APPS_SCRIPT = [...]` 문자열 배열로 한 번 더 감싸져 있습니다(관리자 화면의 "Apps Script 코드 복사" 버튼이 이 문자열을 클립보드에 복사).

이스케이프 규칙이 하나 더 있습니다: JSON 인코딩 시 **`</`로 시작하는 모든 닫는 태그는 `</`로 치환**되어 있습니다(`</script>`가 문자열 안에 그대로 있으면 실제 `<script>` 태그를 조기 종료시키기 때문). 이 규칙을 깨면 파일이 깨집니다.

원본 Claude Design 캔버스(`.dc.html` 아트보드)가 있다면 그쪽에서 편집하고 다시 export하는 게 정석입니다. 캔버스가 없거나 접근할 수 없다면, 아래처럼 **번들을 직접 디코딩 → 수정 → 같은 규칙으로 재인코딩**하는 방법으로 안전하게 고칠 수 있습니다 (Claude Code 세션 몇 번에 걸쳐 실제로 검증된 절차입니다).

## 절차 (Claude Code / 스크립트로 작업할 때)

### 1. 템플릿 디코딩

```python
import json
with open('index.html', 'r', encoding='utf-8') as f:
    content = f.read()

marker = 'type="__bundler/template">'
idx = content.find(marker)
start = idx + len(marker)
end = content.find('</script>', start)
decoded = json.loads(content[start:end].strip())

with open('template.html', 'w', encoding='utf-8') as f:
    f.write(decoded)
```

`template.html`이 실제 페이지 소스입니다. HTML 마크업(`sc-if`, `sc-for`, `sc-camel-on-click` 같은 커스텀 디렉티브가 섞인 템플릿)과, 그 뒤쪽에 `class Component extends DCLogic { ... }` 형태의 실제 JS 로직이 함께 들어 있습니다.

### 2. `template.html`을 수정

일반 텍스트 편집기로 수정하면 됩니다. 기존 코드와 똑같은 패턴(`sc-if`/`sc-for`/`sc-camel-on-click`, `{{ 표현식 }}` 바인딩, `setState` 기반 상태 관리)을 따라가면 됩니다.

### 3. 수정 후 검증 (재인코딩 전에)

- **`sc-if` 짝 확인**: `<sc-if` 개수와 `</sc-if>` 개수가 같아야 합니다.
  ```bash
  grep -c '<sc-if ' template.html
  grep -c '</sc-if>' template.html
  ```
- **JS 문법 확인**: `class Component extends DCLogic {` 부분부터 그 `<script>`가 끝나는 지점까지를 잘라내서 `DCLogic` 스텁 클래스와 함께 `node --check`로 검증합니다.
  ```bash
  # const KEY = ... 로 시작해서 </script> 직전까지 잘라내고
  echo "class DCLogic { setState(o){Object.assign(this.state,o);} }" > component_body.js
  # (그 뒤에 잘라낸 구간을 이어붙인 뒤)
  node --check component_body.js
  ```

### 4. 재인코딩 (원본과 똑같은 이스케이프 규칙으로)

```python
import json
with open('index.html', 'r', encoding='utf-8') as f:
    content = f.read()

marker = 'type="__bundler/template">'
idx = content.find(marker)
start = idx + len(marker)
end = content.find('</script>', start)

with open('template.html', 'r', encoding='utf-8') as f:
    new_decoded = f.read()

new_json = json.dumps(new_decoded, ensure_ascii=False).replace('</', '<\\u002F')
new_block = '\n' + new_json + '\n  '
new_content = content[:start] + new_block + content[end:]

with open('index.html', 'w', encoding='utf-8') as f:
    f.write(new_content)
```

`ensure_ascii=False`가 중요합니다 — 원본은 한글을 `\uXXXX`가 아니라 원문 그대로 담고 있습니다.

### 5. 라운드트립 검증 (필수)

재인코딩한 `index.html`을 다시 디코딩해서, 1번에서 만든 수정된 `template.html`과 **완전히 똑같은지** 바이트 단위로 비교합니다. 다르면 어딘가 이스케이프가 깨진 것이니 절대 커밋하면 안 됩니다.

```python
import json
with open('index.html', 'r', encoding='utf-8') as f:
    content = f.read()
marker = 'type="__bundler/template">'
idx = content.find(marker)
start = idx + len(marker)
end = content.find('</script>', start)
decoded_new = json.loads(content[start:end].strip())

with open('template.html', 'r', encoding='utf-8') as f:
    expected = f.read()

assert decoded_new == expected, "라운드트립 불일치 — 커밋 금지"
```

### 6. 브라우저로 실제 확인

가능하면 로컬에서 `index.html`을 열어(또는 headless 브라우저로) 콘솔 에러 없이 로드되는지, 고친 화면이 실제로 의도대로 동작하는지 확인한 뒤 커밋합니다. 관리자 화면처럼 로그인이 필요한 영역은 `localStorage`에 필요한 키(`gbird-admin-ok` 등)를 직접 넣어 로그인 상태를 흉내 내서 확인할 수 있습니다.

## 백엔드(Apps Script) 코드를 고쳤다면

`template.html` 안의 `const APPS_SCRIPT = [...]` 문자열 배열을 고친 뒤 재인코딩까지 마쳤다면, 그걸로 끝이 아닙니다 — **실제로 배포된 Apps Script에도 새 코드를 붙여넣고 재배포**해야 합니다. 관리자 화면의 "Apps Script 코드 복사"로 새 코드를 꺼내서 [`01-Apps-Script-배포.md`](01-Apps-Script-배포.md)의 재배포 절차를 따르세요.
