# G-Bird 지원서 (GbirdApply)

KAIST 배드민턴 동아리 **G-Bird**의 신입 부원 지원서 웹앱이 배포되는 저장소입니다.

> **2026-09-28부터 지원서는 홈페이지 저장소(`seungjae24/g-bird`)의 `recruit/` 폴더에서 만들고, 홈페이지와 같은 Supabase DB를 씁니다.**
> 이 저장소에는 빌드된 결과물만 올라갑니다(`npm run deploy:recruit`). 여기서 직접 고치지 마세요.
> 사용법·운영 절차·시트 동기화 설정은 홈페이지 저장소의 [`docs/manual/07-신입-모집.md`](https://github.com/seungjae24/g-bird/blob/main/docs/manual/07-신입-모집.md)를 보세요.

- 새 지원서 미리보기: `https://chaeminna.github.io/GbirdApply/next/`
- 실제 지원 주소: `https://chaeminna.github.io/GbirdApply/` — 새 지원서로 전환하기 전까지는 2026 가을학기까지 쓰던 예전 지원서(`index.html`, Google Apps Script + 시트 방식)가 그대로 떠 있습니다. 전환은 `npm run deploy:recruit -- --live`.

## 예전 지원서(2026 가을까지)

루트의 `index.html`(Claude Design 번들, 백엔드는 Apps Script + 스프레드시트)과 `docs/`는 예전 방식의 기록입니다. 새 지원서로 전환한 뒤에는 `index.html`은 git 기록에만 남고, 예전 시즌 지원자 기록은 구글 시트에 그대로 있습니다.

- [`docs/HANDOVER.md`](docs/HANDOVER.md) — 예전 방식 인수인계 요약
- [`docs/setup/02-index-html-직접-수정.md`](docs/setup/02-index-html-직접-수정.md) — 예전 번들 파일을 고칠 때의 절차
