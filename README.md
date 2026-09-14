# G-Bird 지원서 (GbirdApply)

KAIST 배드민턴 동아리 **G-Bird**의 신입 부원 지원서 웹앱입니다. 지원자는 링크 하나로 지원서를 작성/수정하고 면접·참석 여부를 확인하며, 운영진은 같은 화면에서 지원자 목록을 확인하고 면접 시각·합격 여부를 입력합니다.

- 프런트엔드: 이 저장소의 **`index.html` 하나뿐**입니다. 별도의 빌드 과정이 없고, 정적 파일로 아무 곳에나 올려서(GitHub Pages 등) 열면 동작합니다.
- 백엔드/데이터베이스: **Google Apps Script + Google 스프레드시트**. 별도 서버나 DB가 없습니다. 지원자 데이터는 전부 스프레드시트 안에 있습니다.

> ⚠️ `index.html`은 텍스트 에디터로 손 편집하는 파일이 아닙니다. 왜 그런지, 그래도 고쳐야 할 때 어떻게 하는지는 [`docs/setup/02-index-html-직접-수정.md`](docs/setup/02-index-html-직접-수정.md)에 정리해뒀습니다.

## 문서 구조

인수인계·운영에 필요한 내용은 전부 [`docs/`](docs/) 아래에 있습니다. 처음이라면 [`docs/HANDOVER.md`](docs/HANDOVER.md)부터 읽어주세요.

- [`docs/HANDOVER.md`](docs/HANDOVER.md) — 인수인계 시 전체 그림을 잡기 위한 요약. 새 담당자가 가장 먼저 볼 문서.
- [`docs/manual/`](docs/manual/) — 화면별 사용 매뉴얼 (지원자 화면, 관리자 화면)
- [`docs/setup/`](docs/setup/) — Apps Script 배포, `index.html`을 직접 고쳐야 할 때의 절차
- [`docs/integrations/`](docs/integrations/) — Google 스프레드시트 연동 구조(컬럼, API 액션 목록)
- [`docs/IMPROVEMENTS.md`](docs/IMPROVEMENTS.md) — 앞으로 하면 좋을 개선 아이디어 목록
