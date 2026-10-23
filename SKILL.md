---
name: topic-paper-collector
description: "분야·기간(날짜)·키워드·venue 목록에 '특정 주제(예: Multi-agent, LLM, RAG)와 관련된 논문만'이라는 2차 조건을 걸어 논문을 찾고, 공개 PDF를 '논문제목(venue).pdf'로 references 폴더에 내려받는다. ICML·NeurIPS·ICLR·KDD·ACL·TKDE 등 주요 venue와 ICAIF·Journal of Forecasting까지 지원. '분야: … 기간: … 키워드: … Venue: … 의 논문 중에서 ○○ 관련 연구 찾아서 받아줘', '최근 3년 ○○ 논문 PDF 모아줘', '선행연구 references 폴더에 정리해줘' 같은 요청이면 skill 이름을 말하지 않아도 이 스킬을 쓸 것."
---

# 주제 필터 논문 수집 (topic-paper-collector)

요청은 보통 이런 모양이다.

```
분야 : Time-Series Forecasting
기간 : 2024-10-23 ~ 2026-10-23
키워드 : Multi-variate TSF, Agentic Forecasting, Long-term TSF
Venue : ICML, AAAI, ..., ICAIF
→ 이 중에서 Multi-agent system과 관련된 연구를 찾아 references 폴더에 PDF로 받아줘
```

즉 **분야·키워드로 모집단을 만들고, "○○ 관련"이라는 주제 조건으로 걸러낸다.** 이 두 층을 구분하는 것이 이 스킬의 핵심이다. 키워드만으로 검색하면 주제와 무관한 수백 편이 나오고, 주제어만으로 검색하면 분야 밖 논문(예: 로보틱스의 multi-agent planning)이 섞인다.

## 0. 실행 환경

- 필요한 사이트: dblp.org, api.semanticscholar.org, api.openalex.org, export.arxiv.org, arxiv.org, aclanthology.org, openreview.net, proceedings.mlr.press, proceedings.neurips.cc, ijcai.org, jmlr.org, ojs.aaai.org, dl.acm.org.
- 클라우드 작업공간 셸은 보통 이 사이트들이 막혀 있다. 그때는 다른 명령으로 우회하지 말고 **사용자 컴퓨터의 셸(device bash)** 에서 실행한다. 연결된 폴더가 없으면 사용자에게 "+" → "Add folder"로 저장할 폴더를 추가해 달라고 하고, 추가되면 그 폴더 안에서 작업한다.
- 스크립트는 Python 3.8+ 표준 라이브러리만 쓴다. 작업 폴더에 `paper_collect.py`가 없으면 이 문서 맨 아래 **부록** 코드를 그대로 `paper_collect.py`로 저장한다(있으면 재사용). 실행 후 `candidates.json` 같은 중간 파일은 references 밖에 둔다.
- 선택 환경변수: `S2_API_KEY`(Semantic Scholar 속도 제한 완화), `LIT_EMAIL`.
- **셸이 모두 막혀 있을 때(계정 네트워크 허용 목록이 사용자 컴퓨터 셸에도 적용됨)**: 다른 우회는 시도하지 않고 내장 브라우저에서 JavaScript로 같은 일을 한다. dblp.org 탭에서 DBLP API를 `fetch`하고(같은 출처), OpenAlex(`api.openalex.org`)는 CORS가 열려 있어 같은 탭에서 초록(abstract_inverted_index)·DOI·OA PDF 위치를 받는다. 긴 작업은 `window.__job`에 진행 상황을 기록하는 백그라운드 함수로 돌리고 상태만 확인한다(도구 호출 45초 제한). Semantic Scholar는 키 없이 쓰면 429가 잦고, arXiv export API는 CORS가 막혀 있다.
- 이 경우 PDF 파일은 받지 못한다. 대신 `references/_index.csv`(file 열 = 예정 파일명 `논문제목(venue).pdf`, DOI, 공개 PDF 링크)와 `references/README.md` 표를 만들어 사용자 폴더에 쓴다. 사용자가 계정 설정에서 논문 사이트를 허용하면 그때 `download`로 받는다.

## 1. 요청 정리

| 항목 | 요청에서 뽑을 것 | 기본값 |
|---|---|---|
| 분야 | 모집단의 큰 틀 (예: Time-Series Forecasting) | 필수 |
| 키워드 | 분야 안의 세부 주제 | 분야명 그대로 |
| 주제 조건 | "○○ 관련 연구만" 부분 (예: Multi-agent system) | 없으면 필터 없이 진행 |
| 기간 | `--date-from/--date-to` (YYYY-MM-DD). "과거 3년"이면 오늘에서 3년 전 | 최근 3년 |
| venue | 라벨로 변환 (`python paper_collect.py venues`) | `ml,nlp,dm` |
| 파일명 | `논문제목(venue).pdf` | 연도 미포함 |

venue 표기 변환 예: `TMLR/JMLR` → `TMLR,JMLR`, `Journal of Forecasting(Research)` → `Journal of Forecasting`(Wiley, 별칭 `jof`). International Journal of Forecasting은 별도 라벨 `IJF`다. 이름이 애매한 저널은 한 번 확인한다.

정리한 조건을 첫 메시지에 한 줄로 밝힌다(예: "TSF × Multi-agent, 2024-10-23~2026-10-23, 18개 venue로 찾겠습니다").

## 2. 검색어 설계 (교차 검색)

DBLP는 **제목 단어의 AND·접두 매칭**이라 짧은 구(2~4단어)가 잘 듣는다. `agent forecast`는 "Agents … Forecasting"도 잡는다. 주제어 × 분야어를 교차시켜 6~12개 구를 만든다.

예 (Multi-agent × TSF):
- `multi-agent forecasting`, `multi-agent time series`, `multiagent forecasting`
- `agent time series`, `agent forecast`, `agentic forecasting`, `agentic time series`
- `LLM agents forecasting`, `agents temporal reasoning`
- 금융 venue가 있으면: `multi-agent stock prediction`, `agent financial forecasting`, `agent trading prediction`

사용자가 준 키워드(예: Long-term TSF)는 주제어와 한 번씩 엮어 넣고(`agent long-term forecasting`), 그 키워드만으로 검색하지는 않는다 — 주제와 무관한 후보가 수백 편 생긴다.

## 3. 검색 + 날짜·주제 필터

```bash
python paper_collect.py search \
  -k "multi-agent forecasting" -k "agent time series" -k "agent forecast" -k "agentic forecasting" \
  -v ICML,AAAI,IJCAI,KDD,ICLR,NeurIPS,CIKM,ICDM,ACL,EMNLP,NAACL,SIGIR,TPAMI,TMLR,JMLR,TNNLS,TKDE,jof,ICAIF \
  --date-from 2024-10-23 --date-to 2026-10-23 \
  --require "time[- ]?series|forecast|predict" \
  --require "multi[- ]?agent|agentic|LLM[- ]based agents?|agents\b" \
  -o candidates.json
```

- `--require`는 제목+초록에 걸리는 정규식이고, 여러 개면 AND다. 첫째는 분야, 둘째는 주제 조건으로 쓴다. 초록이 없어 확인할 수 없는 논문은 `require: partial-no-abstract`로 남기고 목록 끝(`?` 표시)에 둔다.
- 날짜 판정: 학회 논문은 **학회 개최 월**(`CONF_MONTH` 표), 저널은 Semantic Scholar 출판일, 둘 다 없으면 연도만. 월 단위로 비교하고 경계 월은 포함한다. 예: 시작이 2024-10-23이면 KDD 2024(8월)는 빠지고 CIKM 2024(10월)·NeurIPS 2024(12월)는 들어간다. 아직 열리지 않은 학회(예: 11월 개최인데 오늘이 10월)는 자동으로 빠진다.
- 워크숍·데모·Companion은 제외된다. ICAIF 워크숍도 제외된다.
- `--findings`를 붙이면 Findings of ACL/EMNLP/NAACL도 포함한다(기본은 미포함 — 사용자가 venue에 Findings를 적지 않았으면 넣지 않는다).

## 4. 선별 — 직접 초록을 읽는다

정규식은 1차 거름망일 뿐이다. `candidates.json`의 제목·초록을 읽고 주제 조건에 **실제로** 해당하는지 판단한다. 판단 기준을 먼저 한 줄로 정하고 일관되게 적용한다.

Multi-agent system을 예로 들면:
- 포함: 여러 LLM/에이전트가 역할을 나눠 협업·토론·검증하는 예측 프레임워크, 에이전트 기반 예측 파이프라인(planner/forecaster/critic 등), 멀티에이전트 강화학습으로 예측 모델을 구성·선택하는 연구, 에이전트 시뮬레이션으로 시계열을 예측하는 연구.
- 경계(포함하되 표시): 단일 LLM 에이전트(tool use) 기반 예측 — "agentic"이지만 multi-agent는 아님. 사용자에게 포함 여부를 짚어준다.
- 제외: **multi-agent trajectory/motion forecasting**(보행자·차량 등 여러 '행위자'의 궤적 예측 — 시스템이 아니라 예측 대상이 agent인 경우), agent라는 단어만 나오는 논문, 분야 밖 논문.

사용자가 지켜보고 있으면 표(번호 · 제목 · venue · 날짜 · 판단 한 줄)로 보여주고 받을 번호를 확인한다. "알아서 받아줘"였거나 응답이 없으면 포함 판정 전부를 받는다.

## 5. 다운로드

```bash
python paper_collect.py download -c candidates.json -s 1,3,5-8 -d references
```

- 파일명: `논문제목(venue).pdf`. 제목의 `:`는 ` -`로, 금지 문자(`\ / * ? " < > |`)는 공백으로 바꾸고 150자에서 자른다. `--with-year`면 `(NeurIPS 2025)`.
- PDF 찾는 순서: Semantic Scholar OA PDF → 출판처 규칙(ACL Anthology, OpenReview, PMLR, NeurIPS, IJCAI, JMLR, arXiv) → 랜딩 페이지 `citation_pdf_url` → OpenAlex OA → arXiv 제목 검색.
- 같은 파일이 있으면 건너뛴다. `references/_index.csv`(Excel로 열림)에 file, title, venue, year, authors, citations, status, pdf_source, link가 누적된다. `pdf_source=arXiv`면 출판본이 아닌 프리프린트다.
- 다운로드가 끝나면 `ls references`로 파일 수와 이름을 확인하고, PDF가 아닌 파일(HTML 오류 페이지 등)이 없는지 본다.

## 6. 보고

짧게:
1. 적용한 조건 한 줄(분야 × 주제, 기간, venue 수).
2. 받은 편수와 저장 위치, venue별 편수.
3. 공개 PDF가 없어 못 받은 논문(제목 · venue · DOI 링크) — 학교 도서관 프록시 등으로 직접 받도록.
4. 경계 판정(단일 에이전트 등)과 제외한 유형(예: trajectory forecasting) 한 줄씩.

## 규칙

- 공개 PDF만 받는다. Sci-Hub·LibGen 등 유료 장벽 우회 경로는 쓰지 않는다.
- venue 라벨은 DBLP 판정을 우선한다(Semantic Scholar 표기는 틀릴 때가 있음).
- 요청 사이 1초 이상 쉬고, 429는 스크립트가 재시도한다. 계속 막히면 잠시 후 다시 하거나 `S2_API_KEY`를 안내한다.
- DBLP는 최신 학회(직전 몇 달)를 늦게 반영한다. 기간 끝 무렵 학회가 비어 있으면 그 사실을 보고에 적는다.
- DBLP는 한 페이지에 최대 100건만 준다(`f`로 페이지 넘김). 넓은 주제어(예: `agent`)는 `agent streamid:conf/nips: year:2025:`처럼 venue·연도별로 쪼개 전수 수집한 뒤 제목 정규식으로 거르면 누락이 적다. 요청 간격은 2~4초로 두고 429가 나면 10초 이상 쉰다.
- DBLP는 Findings·Industry Track 논문에도 `ACL`/`EMNLP` venue 문자열을 붙이고, ICDM 워크숍(ICDMW)도 `ICDM`으로 나올 수 있다. **DOI로 판별**한다(`.findings-`, `-industry.`, `icdmw`). 스크립트가 자동 처리한다.

## venue 라벨

| 그룹 | venue |
|---|---|
| ml | ICML, IJCAI, NeurIPS, AAAI, ICLR, TPAMI, TNNLS, TMLR, JMLR, Nature Machine Intelligence, Scientific Reports, Nature Communications |
| nlp | ACL, EMNLP, NAACL, EACL, AACL, COLM, COLING, TACL (+ `--findings`) |
| dm | SIGIR, KDD, CIKM, ICDM, WWW, WSDM, IEEE BigData, TKDE |
| de | SIGMOD, OSDI, ICDE, VLDB, TKDE |
| fin | ICAIF, Journal of Forecasting, IJF |
| extra | AISTATS, UAI, RecSys, ECIR, SDM, PAKDD, ECML-PKDD, TKDD, TOIS, DMKD, EDBT, CIDR, VLDB Journal, TODS |

별칭: nips→NeurIPS, jof·forecasting→Journal of Forecasting, bigdata→IEEE BigData, pvldb→VLDB, ecml→ECML-PKDD.

## 부록: paper_collect.py

```python
#!/usr/bin/env python3
"""paper_collect.py - venue + date-range + topic-filtered paper search, open-access PDF download (stdlib only).

Sources: DBLP (venue-accurate metadata), Semantic Scholar (relevance search, abstracts, dates,
citation counts, OA PDFs), OpenAlex / arXiv / publisher landing pages (PDF fallbacks).

  python paper_collect.py venues
  python paper_collect.py search -k "multi-agent forecasting" -k "agent time series" \
         -v ICML,NeurIPS,ICAIF --date-from 2024-10-23 --date-to 2026-10-23 \
         --require "time[- ]?series|forecast" --require "multi[- ]?agent|agentic|agents?\\b" -o candidates.json
  python paper_collect.py download -c candidates.json -s 1,3,5-8 -d references

Env (optional): S2_API_KEY (Semantic Scholar key, higher rate limit), LIT_EMAIL (polite UA).
"""
import argparse, csv, html, json, os, re, sys, time
import urllib.error, urllib.parse, urllib.request

EMAIL = os.environ.get("LIT_EMAIL", "")
S2_KEY = os.environ.get("S2_API_KEY", "")
UA = "Mozilla/5.0 (paper-collect-skill/1.0" + (f"; mailto:{EMAIL}" if EMAIL else "") + ")"
S2_SEARCH = "https://api.semanticscholar.org/graph/v1/paper/search"
S2_BATCH = "https://api.semanticscholar.org/graph/v1/paper/batch"
S2_FIELDS = ("title,venue,publicationVenue,year,publicationDate,authors,externalIds,openAccessPdf,"
             "citationCount,abstract")

# Month a conference is usually held. A conference paper "appears" at the event, so its date is
# (year, this month); journals use the publication date. Range checks are month-granular and
# the boundary month is included.
CONF_MONTH = {"AAAI": 2, "ICLR": 4, "WWW": 4, "WSDM": 3, "NAACL": 5, "EACL": 3, "ICDE": 5, "SIGMOD": 6,
              "ICML": 7, "SIGIR": 7, "ACL": 7, "IJCAI": 8, "KDD": 8, "VLDB": 8, "COLING": 1, "COLM": 10,
              "CIKM": 10, "EMNLP": 11, "AACL": 11, "ICAIF": 11, "NeurIPS": 12, "ICDM": 12, "IEEE BigData": 12,
              "AISTATS": 5, "UAI": 7, "RecSys": 9, "ECIR": 4, "SDM": 5, "PAKDD": 5, "ECML-PKDD": 9,
              "EDBT": 3, "CIDR": 1, "OSDI": 7}

# label -> ([regexes over DBLP / Semantic Scholar venue strings], Semantic Scholar venue-filter name)
VENUES = {
    # ---- AI / ML
    "ICML": ([r"^ICML$", r"^International Conference on Machine Learning$"], "International Conference on Machine Learning"),
    "IJCAI": ([r"^IJCAI\b", r"International Joint Conference on Artificial Intelligence"], "International Joint Conference on Artificial Intelligence"),
    "NeurIPS": ([r"^(NeurIPS|NIPS)\b", r"Neural Information Processing Systems"], "Neural Information Processing Systems"),
    "AAAI": ([r"^AAAI$", r"^AAAI Conference on Artificial Intelligence$", r"Proceedings of the AAAI Conference"], "AAAI Conference on Artificial Intelligence"),
    "ICLR": ([r"^ICLR\b", r"International Conference on Learning Representations"], "International Conference on Learning Representations"),
    "TPAMI": ([r"Trans\. Pattern Anal\. Mach\. Intell", r"Transactions on Pattern Analysis and Machine Intelligence"], "IEEE Transactions on Pattern Analysis and Machine Intelligence"),
    "TNNLS": ([r"Trans\. Neural Networks Learn\. Syst", r"Transactions on Neural Networks and Learning Systems"], "IEEE Transactions on Neural Networks and Learning Systems"),
    "TMLR": ([r"Trans\. Mach\. Learn\. Res", r"Transactions on Machine Learning Research"], "Transactions on Machine Learning Research"),
    "JMLR": ([r"^J\. Mach\. Learn\. Res", r"^Journal of Machine Learning Research$"], "Journal of Machine Learning Research"),
    "Nature Machine Intelligence": ([r"^Nat\. Mach?\. Intell", r"^Nature Machine Intelligence$"], "Nature Machine Intelligence"),
    "Scientific Reports": ([r"^Sci(entific)?\.? Rep(orts)?\.?$"], "Scientific Reports"),
    "Nature Communications": ([r"^Nat(ure)?\.? Commun(ications)?\.?$"], "Nature Communications"),
    # ---- NLP
    "ACL": ([r"^ACL( \((1|2|Long Papers|Short Papers|Volume \d)\))?$", r"^Annual Meeting of the Association for Computational Linguistics$"], "Annual Meeting of the Association for Computational Linguistics"),
    "EMNLP": ([r"^EMNLP( \(\d\))?$", r"^Conference on Empirical Methods in Natural Language Processing$"], "Conference on Empirical Methods in Natural Language Processing"),
    "NAACL": ([r"^NAACL(-HLT)?( \(\d\))?$", r"North American Chapter of the Association for Computational Linguistics"], "North American Chapter of the Association for Computational Linguistics"),
    "EACL": ([r"^EACL( \(\d\))?$", r"European Chapter of the Association for Computational Linguistics"], "Conference of the European Chapter of the Association for Computational Linguistics"),
    "AACL": ([r"^(AACL|IJCNLP-AACL|AACL[/-]IJCNLP)( \(\d\))?$", r"Asia-Pacific Chapter of the Association for Computational Linguistics"], "AACL"),
    "COLM": ([r"^COLM$", r"Conference on Language Modeling"], "Conference on Language Modeling"),
    "COLING": ([r"^COLING\b", r"International Conference on Computational Linguistics"], "International Conference on Computational Linguistics"),
    "TACL": ([r"Trans\. Assoc\. Comput\. Linguistics", r"Transactions of the Association for Computational Linguistics"], "Transactions of the Association for Computational Linguistics"),
    # ---- Data mining / IR
    "SIGIR": ([r"^SIGIR\b", r"Research and Development in Information Retrieval"], "Annual International ACM SIGIR Conference on Research and Development in Information Retrieval"),
    "KDD": ([r"^KDD\b", r"^Knowledge Discovery and Data Mining$", r"^ACM SIGKDD"], "Knowledge Discovery and Data Mining"),
    "CIKM": ([r"^CIKM\b", r"Conference on Information and Knowledge Management"], "International Conference on Information and Knowledge Management"),
    "ICDM": ([r"^ICDM$", r"^(IEEE )?International Conference on Data Mining$", r"International Conference on Data Mining \(ICDM\)$"], "IEEE International Conference on Data Mining"),
    "WWW": ([r"^WWW\b", r"Web Conference"], "The Web Conference"),
    "WSDM": ([r"^WSDM\b", r"Web Search and Data Mining"], "Web Search and Data Mining"),
    "IEEE BigData": ([r"^(IEEE )?Big ?Data$", r"IEEE International Conference on Big Data"], "BigData"),
    "TKDE": ([r"Trans\. Knowl\. Data Eng", r"Transactions on Knowledge and Data Engineering"], "IEEE Transactions on Knowledge and Data Engineering"),
    # ---- Data engineering / systems
    "SIGMOD": ([r"^SIGMOD(?! Rec)", r"^Proc\. ACM Manag\. Data", r"Proceedings of the ACM on Management of Data"], "SIGMOD Conference"),
    "OSDI": ([r"^OSDI$", r"Operating Systems Design and Implementation"], "USENIX Symposium on Operating Systems Design and Implementation"),
    "ICDE": ([r"^ICDE$", r"International Conference on Data Engineering( \(ICDE\))?$"], "IEEE International Conference on Data Engineering"),
    "VLDB": ([r"^Proc\. VLDB Endow", r"Proceedings of the VLDB Endowment", r"^VLDB$", r"Very Large Data Bases"], "Proceedings of the VLDB Endowment"),
    # ---- Recommended extras (group "extra")
    "AISTATS": ([r"^AISTATS$", r"Artificial Intelligence and Statistics"], "International Conference on Artificial Intelligence and Statistics"),
    "UAI": ([r"^UAI$", r"Uncertainty in Artificial Intelligence"], "Conference on Uncertainty in Artificial Intelligence"),
    "RecSys": ([r"^RecSys$", r"ACM Conference on Recommender Systems"], "ACM Conference on Recommender Systems"),
    "ECIR": ([r"^ECIR\b", r"European Conference on Information Retrieval"], "European Conference on Information Retrieval"),
    "SDM": ([r"^SDM$", r"SIAM International Conference on Data Mining"], "SDM"),
    "PAKDD": ([r"^PAKDD\b", r"Pacific-Asia Conference on Knowledge Discovery"], "Pacific-Asia Conference on Knowledge Discovery and Data Mining"),
    "ECML-PKDD": ([r"^ECML[/ -]?PKDD\b", r"^ECML\b", r"European Conference on Machine Learning"], "ECML/PKDD"),
    "TKDD": ([r"Trans\. Knowl\. Discov\. Data", r"Transactions on Knowledge Discovery from Data"], "ACM Transactions on Knowledge Discovery from Data"),
    "TOIS": ([r"^ACM Trans\. Inf\. Syst", r"^ACM Transactions on Information Systems$"], "ACM Transactions on Information Systems"),
    "DMKD": ([r"Data Min\. Knowl\. Discov", r"^Data Mining and Knowledge Discovery$"], "Data Mining and Knowledge Discovery"),
    "EDBT": ([r"^EDBT\b", r"Extending Database Technology"], "International Conference on Extending Database Technology"),
    "CIDR": ([r"^CIDR$", r"Conference on Innovative Data Systems Research"], "Conference on Innovative Data Systems Research"),
    "VLDB Journal": ([r"^(The )?VLDB J(ournal|\.)?$"], "The VLDB journal"),
    "TODS": ([r"Trans\. Database Syst", r"Transactions on Database Systems"], "ACM Transactions on Database Systems"),
    # ---- Finance / forecasting (group "fin")
    "ICAIF": ([r"^ICAIF$", r"International Conference on (AI|Artificial Intelligence) in Finance$"], "International Conference on AI in Finance"),
    "Journal of Forecasting": ([r"^J(ournal)?\.? (of )?Forecast(ing)?\.?$"], "Journal of Forecasting"),
    "IJF": ([r"^Int(ernational|\.)? ?J(ournal|\.)? (of )?Forecast(ing)?\.?$"], "International Journal of Forecasting"),
}
GROUPS = {
    "ml": ["ICML", "IJCAI", "NeurIPS", "AAAI", "ICLR", "TPAMI", "TNNLS", "TMLR", "JMLR",
           "Nature Machine Intelligence", "Scientific Reports", "Nature Communications"],
    "nlp": ["ACL", "EMNLP", "NAACL", "EACL", "AACL", "COLM", "COLING", "TACL"],
    "dm": ["SIGIR", "KDD", "CIKM", "ICDM", "WWW", "WSDM", "IEEE BigData", "TKDE"],
    "de": ["SIGMOD", "OSDI", "ICDE", "VLDB", "TKDE"],
    "extra": ["AISTATS", "UAI", "RecSys", "ECIR", "SDM", "PAKDD", "ECML-PKDD", "TKDD", "TOIS",
              "DMKD", "EDBT", "CIDR", "VLDB Journal", "TODS"],
    "fin": ["ICAIF", "Journal of Forecasting", "IJF"],
}
ALIASES = {"nips": "NeurIPS", "nmi": "Nature Machine Intelligence", "scirep": "Scientific Reports",
           "natcomm": "Nature Communications", "bigdata": "IEEE BigData", "pvldb": "VLDB",
           "ossi": "OSDI", "ecml": "ECML-PKDD", "vldbj": "VLDB Journal", "jof": "Journal of Forecasting",
           "journal of forecasting research": "Journal of Forecasting", "forecasting": "Journal of Forecasting",
           "international journal of forecasting": "IJF"}
EXCLUDE = re.compile(r"workshop|companion|\(demo|tutorial|doctoral|\(student|\bSRW\b", re.I)
STOP = set("a an the of for and in on with to via from by using based towards toward".split())


def norm(t):
    return re.sub(r"[^a-z0-9]+", "", (t or "").lower())


def venue_label(v):
    v = (v or "").strip()
    if not v or EXCLUDE.search(v):
        return None
    if re.search(r"findings", v, re.I):
        u = v.upper()
        for b in ("EMNLP", "NAACL", "EACL", "AACL", "ACL"):
            if b in u:
                return f"Findings of {b}"
        return "Findings"
    for lab, (pats, _) in VENUES.items():
        if any(re.search(p, v, re.I) for p in pats):
            return lab
    return None


# ---------------------------------------------------------------- HTTP
def get(url, accept="application/json", tries=4, timeout=40, headers=None, data=None):
    h = {"User-Agent": UA, "Accept": accept}
    h.update(headers or {})
    for i in range(tries):
        try:
            req = urllib.request.Request(url, headers=h, data=data)
            with urllib.request.urlopen(req, timeout=timeout) as r:
                return r.read(), r.headers.get("Content-Type", "") or "", r.geturl()
        except urllib.error.HTTPError as e:
            if e.code in (429, 500, 502, 503, 504) and i < tries - 1:
                time.sleep(3 * (2 ** i))
                continue
            raise
        except OSError:
            if i < tries - 1:
                time.sleep(2 ** i)
                continue
            raise


def get_json(url, **kw):
    return json.loads(get(url, **kw)[0].decode("utf-8", "replace"))


def warn(msg):
    print(f"[warn] {msg}", file=sys.stderr)


# ---------------------------------------------------------------- search
def in_years(y, yf, yt):
    return (not yf or (y and y >= yf)) and (not yt or (y and y <= yt))


def dblp_search(q, yf, yt, keep):
    hits, f, total = [], 0, 1
    while f < total and f < 3000:  # DBLP returns at most 100 hits per page
        url = "https://dblp.org/search/publ/api?" + urllib.parse.urlencode({"q": q, "format": "json", "h": 100, "f": f})
        h = get_json(url).get("result", {}).get("hits", {}) or {}
        total = int(h.get("@total", 0) or 0)
        hits += h.get("hit", []) or []
        f += 100
        time.sleep(2)
    out = []
    for h in hits:
        i = h.get("info", {})
        if i.get("type") not in ("Conference and Workshop Papers", "Journal Articles"):
            continue
        vs = i.get("venue")
        vs = vs if isinstance(vs, list) else [vs]
        lab = next((l for l in (venue_label(v) for v in vs) if l), None)
        y = int(i.get("year", 0) or 0)
        if not keep(lab) or not in_years(y, yf, yt):
            continue
        au = (i.get("authors") or {}).get("author", [])
        au = au if isinstance(au, list) else [au]
        names = [re.sub(r" \d{4}$", "", a.get("text", "") if isinstance(a, dict) else str(a)) for a in au]
        ee = i.get("ee", [])
        ee = ee if isinstance(ee, list) else [ee]
        out.append(dict(title=html.unescape(i.get("title", "")).rstrip("."), venue=lab, venue_raw=vs[0], year=y,
                        authors=names, doi=i.get("doi", ""), ee=ee, dblp=i.get("url", ""), src=["dblp"]))
    return out


def s2_paper(p):
    ext = p.get("externalIds") or {}
    pv = p.get("publicationVenue") or {}
    cands = [p.get("venue"), pv.get("name")] + list(pv.get("alternate_names") or [])
    lab = next((l for l in (venue_label(v) for v in cands) if l), None)
    return dict(title=(p.get("title") or "").rstrip("."), venue=lab, venue_raw=p.get("venue") or pv.get("name") or "",
                year=p.get("year") or 0, authors=[a.get("name", "") for a in p.get("authors") or []],
                doi=ext.get("DOI", ""), arxiv=ext.get("ArXiv", ""), acl=ext.get("ACL", ""),
                oa_pdf=(p.get("openAccessPdf") or {}).get("url") or "", abstract=p.get("abstract") or "",
                citations=p.get("citationCount") or 0, s2=p.get("paperId", ""), pubdate=p.get("publicationDate") or "",
                ee=[], src=["s2"])


def s2_search(q, labels, yf, yt, keep, maxn):
    hdr = {"x-api-key": S2_KEY} if S2_KEY else {}
    yr = f"{yf or ''}-{yt or ''}" if (yf or yt) else None
    vf = ",".join(VENUES[l][1] for l in labels if l in VENUES)
    for use_venue in ([True, False] if vf else [False]):
        out, off = [], 0
        while off < min(maxn, 999):
            params = {"query": q, "fields": S2_FIELDS, "limit": 100, "offset": off}
            if yr:
                params["year"] = yr
            if use_venue:
                params["venue"] = vf
            try:
                d = get_json(S2_SEARCH + "?" + urllib.parse.urlencode(params), headers=hdr)
            except urllib.error.HTTPError as e:
                if e.code == 400 and use_venue:
                    break
                raise
            data = d.get("data") or []
            for p in data:
                pp = s2_paper(p)
                if keep(pp["venue"]) and in_years(pp["year"], yf, yt):
                    out.append(pp)
            off += 100
            time.sleep(1.1)
            if len(data) < 100 or d.get("next") is None:
                break
        if out:
            return out
    return []


def s2_enrich(papers):
    hdr = {"x-api-key": S2_KEY} if S2_KEY else {}
    need = [p for p in papers if "s2" not in p["src"] and p.get("doi")]
    for i in range(0, len(need), 400):
        chunk = need[i:i + 400]
        body = json.dumps({"ids": ["DOI:" + p["doi"] for p in chunk]}).encode()
        try:
            res = json.loads(get(S2_BATCH + "?fields=" + S2_FIELDS, data=body,
                                 headers={"Content-Type": "application/json", **hdr}, timeout=90)[0])
        except Exception as e:
            warn(f"S2 enrich failed: {e}")
            continue
        for p, r in zip(chunk, res):
            if not r:
                continue
            s = s2_paper(r)
            for f in ("abstract", "oa_pdf", "arxiv", "acl", "s2", "pubdate"):
                if s.get(f) and not p.get(f):
                    p[f] = s[f]
            p["citations"] = max(p.get("citations") or 0, s["citations"])
        time.sleep(1.1)


def merge(items):
    by = {}
    for p in items:
        k = norm(p["title"])
        if not k:
            continue
        if k not in by:
            by[k] = p
            continue
        q = by[k]
        for f in ("doi", "arxiv", "acl", "oa_pdf", "abstract", "dblp", "s2", "pubdate"):
            if p.get(f) and not q.get(f):
                q[f] = p[f]
        q["ee"] = list(dict.fromkeys((q.get("ee") or []) + (p.get("ee") or [])))
        q["citations"] = max(q.get("citations") or 0, p.get("citations") or 0)
        if "dblp" in p["src"] and "dblp" not in q["src"]:  # DBLP venue/year are authoritative
            q["venue"], q["year"], q["venue_raw"] = p["venue"], p["year"] or q["year"], p["venue_raw"]
        q["src"] = sorted(set(q["src"]) | set(p["src"]))
    return list(by.values())


def resolve_labels(groups, venues):
    lower = {l.lower(): l for l in VENUES}
    labels = []
    for g in [x.strip().lower() for x in (groups or "").split(",") if x.strip()]:
        if g not in GROUPS:
            sys.exit(f"unknown group '{g}'. groups: {', '.join(GROUPS)}")
        labels += GROUPS[g]
    for v in [x.strip() for x in (venues or "").split(",") if x.strip()]:
        lab = lower.get(v.lower()) or ALIASES.get(v.lower())
        if not lab:
            sys.exit(f"unknown venue '{v}'. run `venues` to list labels")
        labels.append(lab)
    return list(dict.fromkeys(labels))


S2_MATCH = "https://api.semanticscholar.org/graph/v1/paper/search/match"


def s2_match_missing(papers, limit=300):
    """Fill abstracts/dates for papers S2 never returned (e.g. DBLP-only OpenReview papers) by title match."""
    hdr = {"x-api-key": S2_KEY} if S2_KEY else {}
    need = [p for p in papers if not p.get("abstract")][:limit]
    for p in need:
        try:
            d = get_json(S2_MATCH + "?" + urllib.parse.urlencode({"query": p["title"], "fields": S2_FIELDS}),
                         headers=hdr, tries=2)
        except Exception:
            time.sleep(1.1)
            continue
        r = (d.get("data") or [None])[0]
        if r and norm(r.get("title")) == norm(p["title"]):
            s = s2_paper(r)
            for f in ("abstract", "oa_pdf", "arxiv", "acl", "s2", "pubdate", "doi"):
                if s.get(f) and not p.get(f):
                    p[f] = s[f]
            p["citations"] = max(p.get("citations") or 0, s["citations"])
        time.sleep(1.1)


def parse_date(s):
    """'2024-10-23' / '2024-10' / '2024' -> (y, m) or None."""
    if not s:
        return None
    m = re.match(r"^(\d{4})(?:-(\d{1,2}))?", str(s))
    return (int(m.group(1)), int(m.group(2) or 0)) if m else None


def paper_date(p):
    """Return ((year, month), basis). Conferences: event month; journals: S2 publication date; else year only."""
    y = p.get("year") or 0
    if p.get("venue") in CONF_MONTH and y:
        return (y, CONF_MONTH[p["venue"]]), "conference-month"
    d = parse_date(p.get("pubdate"))
    if d and d[1] and (not y or abs(d[0] - y) <= 1):
        return d, "publication-date"
    return ((y, 0), "year-only") if y else (None, "unknown")


def in_range(ym, lo, hi):
    """Month-granular, inclusive. Month 0 (= year only) matches any month of that year."""
    if ym is None:
        return False
    y, m = ym
    if lo and (y < lo[0] or (y == lo[0] and m and lo[1] and m < lo[1])):
        return False
    if hi and (y > hi[0] or (y == hi[0] and m and hi[1] and m > hi[1])):
        return False
    return True


def cmd_search(a):
    lo, hi = parse_date(a.date_from), parse_date(a.date_to)
    yf = a.year_from or (lo[0] if lo else None)
    yt = a.year_to or (hi[0] if hi else None)
    groups = a.groups if a.groups is not None else ("" if a.venues else "ml,nlp,dm,de")
    labels = resolve_labels(groups, a.venues)
    keep = lambda lab: bool(lab) and (lab in labels or (a.findings and lab.startswith("Findings")))
    found = []
    for k in a.keyword:
        try:
            r = dblp_search(k, yf, yt, keep)
            print(f"[dblp] '{k}': {len(r)}", file=sys.stderr)
            found += r
        except Exception as e:
            warn(f"DBLP failed for '{k}': {e}")
        time.sleep(1)
        if not a.no_s2:
            try:
                r = s2_search(k, labels, yf, yt, keep, a.s2_max)
                print(f"[s2]   '{k}': {len(r)}", file=sys.stderr)
                found += r
            except Exception as e:
                warn(f"Semantic Scholar failed for '{k}': {e}")
    papers = merge(found)
    # DBLP labels Findings / workshop papers with the main venue string; the DOI tells them apart.
    fixed = []
    for p in papers:
        d = (p.get("doi") or "").lower()
        if re.search(r"icdmw|workshop", d):
            continue
        m = re.search(r"\.findings-(acl|emnlp|naacl|eacl|aacl)\.", d)
        if m:
            p["venue"] = f"Findings of {m.group(1).upper()}"
            if not a.findings:
                continue
        if re.search(r"-(industry|demo|srw)\.", d):
            continue
        fixed.append(p)
    papers = fixed
    print(f"[merge] {len(papers)} unique papers in venues/years", file=sys.stderr)

    # date range (month-granular)
    kept = []
    for p in papers:
        ym, basis = paper_date(p)
        p["date"] = f"{ym[0]}-{ym[1]:02d}" if ym and ym[1] else (str(ym[0]) if ym else "")
        p["date_basis"] = basis
        if (lo or hi) and not in_range(ym, lo, hi):
            continue
        kept.append(p)
    papers = kept
    print(f"[date]  {len(papers)} within {a.date_from or yf}..{a.date_to or yt}", file=sys.stderr)

    if not a.no_s2 and papers:
        s2_enrich(papers)
        if a.require:
            s2_match_missing(papers)
        for p in papers:  # journals may gain a precise date from S2
            ym, basis = paper_date(p)
            if basis == "publication-date":
                p["date"], p["date_basis"] = f"{ym[0]}-{ym[1]:02d}", basis
        papers = [p for p in papers if not (lo or hi) or in_range(parse_date(p["date"]), lo, hi)]

    # topic filter: every --require regex must match title+abstract
    reqs = [re.compile(r, re.I) for r in (a.require or [])]
    if reqs:
        out = []
        for p in papers:
            text = p["title"] + " " + (p.get("abstract") or "")
            hits = [bool(r.search(text)) for r in reqs]
            if all(hits):
                p["require"] = "matched"
                out.append(p)
            elif not p.get("abstract") and any(r.search(p["title"]) for r in reqs):
                p["require"] = "partial-no-abstract"   # cannot verify; keep for manual review
                out.append(p)
        papers = out
        print(f"[require] {len(papers)} pass topic filter", file=sys.stderr)

    terms = {w for k in a.keyword for w in re.findall(r"[a-z0-9]+", k.lower()) if w not in STOP and len(w) > 2}
    for p in papers:
        t, ab = p["title"].lower(), (p.get("abstract") or "").lower()
        p["score"] = sum(2 if w in t else (1 if w in ab else 0) for w in terms)
    papers.sort(key=lambda p: (p.get("require") == "partial-no-abstract", -p["score"],
                               -(p.get("citations") or 0), -(p.get("year") or 0)))
    for n, p in enumerate(papers, 1):
        p["no"] = n
    meta = dict(keywords=a.keyword, venues=labels, findings=a.findings, date_from=a.date_from, date_to=a.date_to,
                year_from=yf, year_to=yt, require=a.require, created=time.strftime("%Y-%m-%d %H:%M"))
    with open(a.out, "w", encoding="utf-8") as f:
        json.dump({"meta": meta, "papers": papers}, f, ensure_ascii=False, indent=1)
    print(f"{len(papers)} candidates -> {a.out}")
    for p in papers[:a.show]:
        flag = " ?" if p.get("require") == "partial-no-abstract" else ""
        print(f"{p['no']:>4} | {p['venue']} {p['date']} | cit {p.get('citations') or 0:>5} | {p['title'][:110]}{flag}")


# ---------------------------------------------------------------- download
BAD = re.compile(r'[\\/:*?"<>|\x00-\x1f]')


def filename(title, label, year, with_year):
    t = BAD.sub(" ", title.replace(":", " -"))
    t = re.sub(r"\s+", " ", t).strip().rstrip(".")
    t = t[:150].rstrip()
    v = f"{label} {year}" if with_year and year else label
    return f"{t}({v}).pdf"


def ee_pdf(u):
    m = re.match(r"https?://aclanthology\.org/([^/?#]+?)(\.pdf)?/?$", u)
    if m:
        return f"https://aclanthology.org/{m.group(1)}.pdf"
    m = re.search(r"openreview\.net/forum\?id=([^&#]+)", u)
    if m:
        return f"https://openreview.net/pdf?id={m.group(1)}"
    m = re.match(r"(https?://proceedings\.mlr\.press/v\d+)/([^/]+)\.html$", u)
    if m:
        return f"{m.group(1)}/{m.group(2)}/{m.group(2)}.pdf"
    if re.search(r"(proceedings\.neurips\.cc|papers\.nips\.cc).*/hash/.*-Abstract.*\.html$", u):
        return u.replace("/hash/", "/file/").replace("-Abstract", "-Paper").replace(".html", ".pdf")
    if re.search(r"papers\.nips\.cc/paper/\d+-[^/]+$", u):
        return u + ".pdf"
    m = re.match(r"(https?://(www\.)?ijcai\.org/proceedings/\d{4}/\d+)/?$", u)
    if m:
        return m.group(1) + ".pdf"
    m = re.match(r"https?://(www\.)?jmlr\.org/papers/v(\d+)/([^/]+)\.html$", u)
    if m:
        return f"https://jmlr.org/papers/volume{m.group(2)}/{m.group(3)}/{m.group(3)}.pdf"
    m = re.search(r"arxiv\.org/abs/([^?#]+)", u)
    if m:
        return f"https://arxiv.org/pdf/{m.group(1)}"
    return None


def openalex_pdfs(doi):
    if not doi:
        return []
    u = "https://api.openalex.org/works/https://doi.org/" + urllib.parse.quote(doi, safe="/")
    if EMAIL:
        u += "?mailto=" + urllib.parse.quote(EMAIL)
    try:
        d = get_json(u, tries=2)
    except Exception:
        return []
    locs = [d.get("best_oa_location") or {}] + list(d.get("locations") or [])
    return [l["pdf_url"] for l in locs if l and l.get("pdf_url")]


def arxiv_by_title(title):
    q = 'ti:"' + re.sub(r"[^\w\s-]", " ", title).strip() + '"'
    u = "http://export.arxiv.org/api/query?" + urllib.parse.urlencode({"search_query": q, "max_results": 5})
    try:
        s = get(u, accept="application/atom+xml", tries=2)[0].decode("utf-8", "replace")
    except Exception:
        return []
    for e in re.findall(r"<entry>(.*?)</entry>", s, re.S):
        t = re.search(r"<title>(.*?)</title>", e, re.S)
        i = re.search(r"<id>https?://arxiv\.org/abs/([^<]+)</id>", e)
        if t and i and norm(html.unescape(t.group(1))) == norm(title):
            return [f"https://arxiv.org/pdf/{i.group(1)}"]
    return []


def try_pdf(url):
    try:
        data, ct, final = get(url, accept="application/pdf,text/html;q=0.8,*/*;q=0.5", tries=2, timeout=60)
    except Exception:
        return None, None
    if data[:5] == b"%PDF-":
        return data, None
    page = data[:500000].decode("utf-8", "replace") if ("html" in ct.lower() or data.lstrip()[:1] == b"<") else ""
    m = (re.search(r'<meta[^>]+name=["\']citation_pdf_url["\'][^>]+content=["\']([^"\']+)', page, re.I)
         or re.search(r'<meta[^>]+content=["\']([^"\']+)["\'][^>]+name=["\']citation_pdf_url', page, re.I))
    return None, (urllib.parse.urljoin(final, html.unescape(m.group(1))) if m else None)


def fetch_pdf(p, max_tries=10):
    queue = [p.get("oa_pdf")]
    queue += [ee_pdf(u) for u in p.get("ee") or []]
    if p.get("acl"):
        queue.append(f"https://aclanthology.org/{p['acl']}.pdf")
    if p.get("arxiv"):
        queue.append(f"https://arxiv.org/pdf/{p['arxiv']}")
    queue += list(p.get("ee") or [])  # landing pages -> citation_pdf_url
    queue = [u for u in queue if u]
    seen, fallback_done, n = set(), False, 0
    while n < max_tries:
        if not queue:
            if fallback_done:
                break
            fallback_done = True
            queue = openalex_pdfs(p.get("doi")) + arxiv_by_title(p["title"])
            continue
        u = queue.pop(0)
        if u in seen:
            continue
        seen.add(u)
        n += 1
        data, nxt = try_pdf(u)
        if data:
            return data, u
        if nxt and nxt not in seen:
            queue.insert(0, nxt)
        time.sleep(0.5)
    return None, None


def parse_sel(s, n):
    s = (s or "all").strip().lower()
    if s == "all":
        return list(range(1, n + 1))
    if s.startswith("top"):
        return list(range(1, min(n, int(s[3:])) + 1))
    out = []
    for part in s.split(","):
        part = part.strip()
        if "-" in part:
            x, y = part.split("-")
            out += range(int(x), int(y) + 1)
        elif part:
            out.append(int(part))
    return [i for i in dict.fromkeys(out) if 1 <= i <= n]


def cmd_download(a):
    papers = json.load(open(a.candidates, encoding="utf-8"))["papers"]
    bynum = {p["no"]: p for p in papers}
    sel = parse_sel(a.select, max(bynum) if bynum else 0)
    os.makedirs(a.dir, exist_ok=True)
    idx = os.path.join(a.dir, "_index.csv")
    new = not os.path.exists(idx)
    ok, fail = 0, []
    with open(idx, "a", newline="", encoding="utf-8-sig" if new else "utf-8") as f:
        w = csv.writer(f)
        if new:
            w.writerow(["file", "title", "venue", "year", "authors", "citations", "status", "pdf_source", "link", "added"])
        for i in sel:
            p = bynum.get(i)
            if not p:
                continue
            fn = filename(p["title"], p["venue"], p.get("year"), a.with_year)
            path = os.path.join(a.dir, fn)
            au = ", ".join(p.get("authors", [])[:3]) + (" et al." if len(p.get("authors", [])) > 3 else "")
            link = f"https://doi.org/{p['doi']}" if p.get("doi") else (p.get("ee") or [p.get("dblp", "")])[0]
            if os.path.exists(path):
                print(f"= exists  {fn}")
                continue
            data, src = fetch_pdf(p)
            if data:
                with open(path, "wb") as out:
                    out.write(data)
                kind = "arXiv" if "arxiv.org" in src else "publisher/OA"
                w.writerow([fn, p["title"], p["venue"], p.get("year"), au, p.get("citations") or 0, "downloaded", kind, link,
                            time.strftime("%Y-%m-%d")])
                ok += 1
                print(f"+ saved   {fn}  [{kind}]")
            else:
                w.writerow(["", p["title"], p["venue"], p.get("year"), au, p.get("citations") or 0, "no-open-pdf", "", link,
                            time.strftime("%Y-%m-%d")])
                fail.append((p, link))
                print(f"- no PDF  {p['title'][:90]} ({p['venue']})")
            f.flush()
            time.sleep(1)
    print(f"\n{ok} downloaded, {len(fail)} without open-access PDF -> {a.dir}")
    for p, link in fail:
        print(f"  * {p['title']} ({p['venue']} {p.get('year')}) {link}")


def cmd_venues(_):
    for g, labs in GROUPS.items():
        print(f"[{g}] " + ", ".join(labs))
    print("aliases: " + ", ".join(f"{k}->{v}" for k, v in ALIASES.items()))


def main():
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    sub = ap.add_subparsers(dest="cmd", required=True)
    sub.add_parser("venues").set_defaults(func=cmd_venues)
    s = sub.add_parser("search")
    s.add_argument("-k", "--keyword", action="append", required=True, help="search phrase (repeatable)")
    s.add_argument("-g", "--groups", default=None, help="ml,nlp,dm,de,extra,fin (default: ml,nlp,dm,de)")
    s.add_argument("-v", "--venues", default="", help="comma-separated venue labels (alone = only these)")
    s.add_argument("--findings", action="store_true", help="also keep Findings of ACL/EMNLP/NAACL/EACL/AACL")
    s.add_argument("--date-from", help="YYYY-MM-DD (or YYYY-MM / YYYY); month-granular, inclusive")
    s.add_argument("--date-to", help="YYYY-MM-DD (or YYYY-MM / YYYY)")
    s.add_argument("--year-from", type=int)
    s.add_argument("--year-to", type=int)
    s.add_argument("--require", action="append", help="regex that title+abstract must match (repeatable = AND)")
    s.add_argument("--s2-max", type=int, default=300, help="max Semantic Scholar results per phrase")
    s.add_argument("--no-s2", action="store_true", help="DBLP only")
    s.add_argument("--show", type=int, default=40)
    s.add_argument("-o", "--out", default="candidates.json")
    s.set_defaults(func=cmd_search)
    d = sub.add_parser("download")
    d.add_argument("-c", "--candidates", default="candidates.json")
    d.add_argument("-s", "--select", default="all", help="'all', 'top20', or '1,3,5-8' (candidate numbers)")
    d.add_argument("-d", "--dir", default="references")
    d.add_argument("--with-year", action="store_true", help="name files 'Title(Venue YYYY).pdf'")
    d.set_defaults(func=cmd_download)
    a = ap.parse_args()
    a.func(a)


if __name__ == "__main__":
    main()
```
