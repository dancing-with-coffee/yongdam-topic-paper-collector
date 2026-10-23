# topic-paper-collector

분야·기간(날짜)·키워드·venue 목록에 **"○○ 관련 논문만"** 이라는 2차 주제 조건을 걸어 논문을 찾고, 공개 PDF를 `논문제목(venue).pdf`로 `references/` 폴더에 내려받는 Claude skill입니다. PDF를 받을 수 없는 환경에서는 DOI·공개 PDF 링크 표(`_index.csv`, `README.md`)로 정리합니다.

## 요청 예시

```
분야 : Time-Series Forecasting
기간 : 2024-10-23 ~ 2026-10-23
키워드 : Multi-variate TSF, Agentic Forecasting, Long-term TSF
Venue : ICML, AAAI, IJCAI, KDD, ICLR, NeurIPS, CIKM, ICDM, ACL, EMNLP, NAACL, SIGIR, TPAMI, TMLR/JMLR, TNNLS, TKDE, Journal of Forecasting, ICAIF
→ 이 중 Multi-agent system 관련 연구를 references 폴더에 "논문제목(venue).pdf"로 받아줘
```

## 특징

- **2층 필터**: 분야·키워드로 후보를 모으고, `--require` 정규식(제목+초록) → 초록 판독으로 주제 조건을 거른다. agent가 예측 대상인 multi-agent trajectory forecasting 등은 제외.
- **날짜 단위 기간**: 학회 논문은 개최 월, 저널은 출판일 기준(월 단위, 경계 월 포함).
- **venue**: ML·NLP·DM·DB 주요 venue + ICAIF, Journal of Forecasting, International Journal of Forecasting.
- **Findings·워크숍 판별**: DBLP venue 문자열 대신 DOI(`.findings-`, `-industry.`, `icdmw`)로 구분.
- **소스**: DBLP(venue 판정) + Semantic Scholar + OpenAlex + arXiv. 공개(Open Access) PDF만 받는다.

## 구성

| 경로 | 내용 |
|---|---|
| `SKILL.md` | 스킬 본문(스크립트가 부록으로 포함되어 단독으로 동작) |
| `scripts/paper_collect.py` | 검색·필터·다운로드 스크립트 (Python 3.8+, 표준 라이브러리만) |
| `evals/` | 테스트 케이스 (`claude plugin eval` 형식) |

## 스크립트 사용법

```bash
python scripts/paper_collect.py venues
python scripts/paper_collect.py search -k "multi-agent forecasting" -k "agent time series" \
  -v ICML,NeurIPS,ICLR,KDD,ICAIF,jof --date-from 2024-10-23 --date-to 2026-10-23 \
  --require "time[- ]?series|forecast|predict" --require "multi[- ]?agent|agentic|agents\b" -o candidates.json
python scripts/paper_collect.py download -c candidates.json -s all -d references
```

선택 환경변수: `S2_API_KEY`(Semantic Scholar 속도 제한 완화), `LIT_EMAIL`.
