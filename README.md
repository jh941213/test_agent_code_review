
## 📑 목차
1. [프로젝트 개요](#-프로젝트-개요)
2. [전체 아키텍처](#-전체-아키텍처)
3. [API 엔드포인트 상세](#-api-엔드포인트-상세)
4. [핵심 비즈니스 로직](#-핵심-비즈니스-로직)
5. [데이터 플로우](#-데이터-플로우)
6. [코드 품질 분석](#-코드-품질-분석)
7. [개선 권장사항](#-개선-권장사항)

---

## 🎯 프로젝트 개요

### 목적
AI 기반 테스트 자동화 시스템으로, 요구사항 문서로부터 **테스트 시나리오**와 **테스트 케이스**를 자동 생성합니다.

### 주요 기능
- ✅ **테스트 시나리오 자동 생성**: 요구사항 → AI → 테스트 시나리오
- ✅ **테스트 케이스 자동 생성**: 시나리오 → AI → 상세 테스트 케이스
- ✅ **파일 기반 입력**: PDF, Excel, Word 등 다양한 문서 포맷 지원
- ✅ **CSV/Excel 임포트/익스포트**: 기존 테스트 케이스 가져오기/내보내기
- ✅ **버전 관리**: 시맨틱 버저닝을 통한 테스트 데이터 이력 관리

### 기술 스택
```yaml
Backend: Python 3.x, Flask-RESTful
ORM: SQLAlchemy
LLM: AionuLLMClient (Custom LLM API)
파일 변환: MarkItDown (PDF/Excel → Markdown)
병렬 처리: ThreadPoolExecutor (5 workers)
인증: JWT (current_user)
```

---

## 🏗️ 전체 아키텍처

### 계층 구조
```
┌─────────────────────────────────────────────────────────────┐
│                    Controller Layer                          │
│  (Flask-RESTful API Endpoints)                              │
│  - test_scenario_controller.py (시나리오 API)               │
│  - test_case_controller.py (케이스 API)                     │
│  - test_export_controller.py (CSV/Excel 다운로드)           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Service Layer                             │
│  (비즈니스 로직 & DB 처리)                                   │
│  - test_scenario_service.py (1,347 lines)                   │
│  - test_case_service.py (1,417 lines)                       │
│  - file_conversion_service.py (678 lines)                   │
│  - version_service.py (버전 관리)                            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Agent/Workflow Layer                      │
│  (LLM 호출 & 병렬 처리)                                      │
│  - workflow.py (시나리오/케이스 생성 워크플로우)             │
│  - agent.py (LLM 에이전트 베이스)                           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                External LLM API (AionuLLMClient)             │
└─────────────────────────────────────────────────────────────┘
```

### 폴더 구조
```
swe/test_agent/
├── controller/                    # 🌐 API 엔드포인트
│   ├── test_scenario_controller.py   (690 lines)
│   ├── test_case_controller.py       (565 lines)
│   ├── test_export_controller.py     (549 lines)
│   ├── test_plan_controller.py
│   └── test_agent_common_controller.py
│
├── services/                      # 💼 비즈니스 로직
│   ├── test_scenario_service.py      (1,347 lines) ⚠️ God Class
│   ├── test_case_service.py          (1,417 lines) ⚠️ God Class
│   ├── file_conversion_service.py    (678 lines)
│   ├── version_service.py            (224 lines)
│   └── test_agent_file_service.py
│
├── agents/                        # 🤖 LLM 워크플로우
│   ├── workflow.py                   (909 lines)
│   └── agent.py                      (229 lines)
│
├── model_dto/                     # 📦 데이터 모델
│   └── test_agent_dto.py             (Pydantic 모델)
│
├── utils/                         # 🛠️ 유틸리티
│   └── statistics_utils.py
│
└── docs/                          # 📚 문서
    ├── test_agent_api.md             (749 lines)
    └── test_agent_v2_postman.json    (1,941 lines)
```

---

## 🌐 API 엔드포인트 상세

### 1. 테스트 시나리오 API

#### 1.1 시나리오 생성 (Generate)
```http
POST /test-agent/scenario/generate
Content-Type: multipart/form-data
```

**요청 파라미터**:
```json
{
  "project_id": "프로젝트 ID (필수)",
  "project_module_id": "프로젝트 모듈 ID (필수)",
  "user_input": "사용자 입력 텍스트 (선택)",
  "requirement_agent_result_id": "요구사항 분석 결과 ID (선택)",
  "files": "파일 업로드 (선택)",
  "repo_ids": ["저장소 파일 ID 배열 (선택)"],
  "repo_keys": ["저장소 파일 키 배열 (선택)"]
}
```

**입력 소스 상세 설명**:

> 💡 **중요**: 3가지 입력 소스(`user_input`, `requirement_agent_result_id`, `files`)는 **독립적으로 사용 가능**합니다.
> - 1개만 사용 가능 (예: user_input만)
> - 2개 조합 가능 (예: user_input + files)
> - 3개 모두 사용 가능 (모든 입력 통합)
> - 시스템이 자동으로 통합하여 LLM에 전달

**1) user_input** (직접 텍스트 입력)
```python
# 예시
{
  "user_input": "온라인 쇼핑몰 로그인 기능\n- 이메일/비밀번호 로그인\n- SNS 로그인 지원"
}

# 처리 방식
user_input = scenario_input.user_input or ""
# → LLM에게 그대로 전달
```

**2) requirement_agent_result_id** (요구사항 분석 결과 참조)
```python
# 예시
{
  "requirement_agent_result_id": "ac_uuid_1234"  // AgentContent ID
}

# 처리 방식 (file_conversion_service.py:267-348)
# Step 1: DB에서 AgentContent 조회
agent_content = AgentContent.query.filter_by(id='ac_uuid_1234').first()

# Step 2: output_json에서 요구사항 추출
output_json = agent_content.output_json
# {
#   "requirements": [
#     {"id": "REQ_001", "title": "로그인 기능", "description": "..."},
#     {"id": "REQ_002", "title": "회원가입 기능", "description": "..."}
#   ]
# }

# Step 3: Markdown 형식으로 변환
requirement_result = """
# 📋 요구사항 분석 결과

## 📋 요구사항 목록

- **REQ_001**: 로그인 기능
  - 설명: 사용자가 이메일/비밀번호로 로그인
  - 분류: 사용자 관리 > 인증 > 로그인

- **REQ_002**: 회원가입 기능
  - 설명: 신규 사용자 가입 처리
  - 분류: 사용자 관리 > 인증 > 회원가입
"""
```

**3) files** (파일 업로드 - 3가지 방식)

**3-1. 직접 업로드 (Form-data files)**
```python
# Postman/API 클라이언트에서
POST /test-agent/scenario/generate
Content-Type: multipart/form-data

files: [요구사항.pdf]  // 실제 파일 바이너리
files: [테스트명세.xlsx]

# 처리 방식
for file in scenario_input.files:
    upload_repo = self.file_service.create_file(file, "test_agent", user_id)
    # → UploadFile 테이블에 저장
    # → storage에 바이너리 저장
    # → MarkItDown으로 Markdown 변환
```

**3-2. repo_ids (저장된 파일 ID 참조)**
```python
# 예시
{
  "repo_ids": ["repo_uuid_1", "repo_uuid_2"]
}

# 처리 방식 (file_conversion_service.py:96-176)
for repo_id in repo_ids:
    # Step 1: Repository 테이블 조회
    repo_item = Repository.query.filter_by(id=repo_id).first()
    # {
    #   "id": "repo_uuid_1",
    #   "repo_name": "요구사항.pdf",
    #   "repo_path": "2024/01/abc123.pdf",
    #   "upload_file_id": "upload_uuid_1"
    # }

    # Step 2: 파일 로드 (3가지 시도)
    # 시도 1: 로컬 경로에서 직접 읽기
    file_path = os.path.join(storage_local_path, repo_item.repo_path)
    with open(file_path, 'rb') as f:
        file_data = f.read()

    # 시도 2: UploadFile 테이블 → storage 키로 로드
    upload_file = UploadFile.query.filter_by(id=repo_item.upload_file_id).first()
    file_data = storage.load(upload_file.key)

    # 시도 3: repo_path를 storage 키로 직접 사용
    file_data = storage.load(repo_item.repo_path)

    # Step 3: Markdown 변환
    markdown_text = self.markitdown.convert_stream(BytesIO(file_data))
```

**3-3. repo_keys (Storage 직접 키 참조)**
```python
# 예시
{
  "repo_keys": ["2024/01/abc123.pdf", "2024/01/def456.xlsx"]
}

# 처리 방식 (file_conversion_service.py:178-211)
for repo_key in repo_keys:
    # Storage에서 직접 로드 (가장 빠름)
    raw_data = storage.load(repo_key)  # S3, local storage 등

    # Markdown 변환
    markdown_text = self.markitdown.convert_stream(BytesIO(raw_data))
```

**파일 변환 프로세스 (MarkItDown)**:
```python
# file_conversion_service.py:459-552
def convert_bytes_to_markdown(file_data, filename):
    # 지원 포맷: PDF, Word(docx), Excel(xlsx/xls), CSV, TXT, Markdown

    # 1차 시도: In-memory 변환 (빠름)
    bytes_stream = io.BytesIO(file_data)
    result = self.markitdown.convert_stream(
        bytes_stream,
        file_extension='.pdf'  # 확장자 기반 파서 선택
    )
    return result.text_content

    # 2차 시도: 임시 파일 변환 (폴백)
    with tempfile.NamedTemporaryFile(suffix='.pdf') as temp:
        temp.write(file_data)
        result = self.markitdown.convert(temp.name)
        return result.text_content
```

**최종 통합 입력**:
```python
# test_scenario_service.py:187-268
# 모든 입력 소스를 하나로 통합
combined_input = ""

if user_input:
    combined_input += f"사용자 요구사항:\n{user_input}\n\n"

if requirement_result:  # requirement_agent_result_id로 조회한 결과
    combined_input += f"{requirement_result}\n\n"

if file_document:  # repo_ids + repo_keys + files 변환 결과
    combined_input += f"# 📄 참조 파일\n\n{file_document}\n\n"

# → 최종적으로 LLM에 전달되는 입력
# combined_input: "사용자 요구사항:\n...\n\n# 📋 요구사항 분석 결과\n...\n\n# 📄 참조 파일\n..."
```

**응답 예시**:
```json
{
  "scenarios": [
    {
      "sno": 1,
      "scenario_id": "TS_001",
      "scenario_name": "사용자 로그인 시나리오",
      "scenario_description": "사용자가 ID/PW로 로그인",
      "category": {
        "category": "인증/인가",
        "category_abbreviation": "AUTH"
      },
      "priority": "high",
      "test_type": "functional"
    }
  ],
  "total_count": 15,
  "is_saved": false,        // ⭐ 아직 DB에 저장 안됨
  "can_save": true,         // ⭐ 저장 가능
  "input_data": {           // ⭐ 저장 시 필요한 원본 데이터
    "user_input": "...",
    "requirement_result": "...",
    "file_document": "...",
    "repo_ids": [...],
    "repo_keys": [...]
  }
}
```

**처리 흐름**:
```
1. 입력 통합 (user_input + requirement_result + files)
   └─ file_conversion_service.py:68
       ├─ user_input: 그대로 사용
       ├─ requirement_agent_result_id → DB 조회 → Markdown
       └─ repo_ids/repo_keys/files → storage 로드 → Markdown

2. 파일 변환 (PDF/Excel → Markdown)
   └─ MarkItDown 라이브러리 사용
       ├─ convert_stream() - In-memory (빠름)
       └─ convert() - Temp file (폴백)

3. LLM 워크플로우 실행
   └─ workflow.scenario_workflow(combined_input)
       ├─ planner_agent() - 카테고리 분류
       ├─ ThreadPoolExecutor (5 workers)
       └─ scenario_agent() - 병렬 시나리오 생성

4. 결과 반환 (미저장 상태)
   └─ is_saved=false, can_save=true
```

**코드 위치**: `test_scenario_controller.py:86-166`

---

#### 1.2 시나리오 저장 (Save)
```http
POST /test-agent/scenario
Content-Type: application/json
```

**요청 파라미터**:
```json
{
  "scenario_data": {
    "scenarios": [...],      // generate에서 받은 결과
    "input_data": {...},     // generate에서 받은 input_data
    "total_count": 15
  }
}
```

**응답 예시**:
```json
{
  "project_module_id": "uuid-1234",
  "agent_content_id": "uuid-5678",
  "version": "1.0.0",
  "status": "completed",
  "is_saved": true,         // ⭐ DB 저장 완료
  "can_proceed": true,      // ⭐ 다음 단계(케이스 생성) 진행 가능
  "message": "테스트 시나리오가 저장되었습니다."
}
```

**버전 관리 상세 (Save 시점)**:

버전은 **저장(Save) 시점**에 결정됩니다. Generate는 버전과 무관하게 LLM 결과만 반환합니다.

```python
# version_service.py:70-196
def set_version(
    project_id: str,
    project_module_id: str,
    current_version: str = None,
    operation_type: str = 'create',  # 'create' or 'update'
    scope: str = 'module'             # 'module' or 'project'
) -> str:
    """버전 결정 로직"""

    # 1. operation_type = 'create' (신규 생성)
    if operation_type == 'create':
        # 프로젝트 내 모든 test_agent 버전 조회
        existing_versions = ["1.0.0", "2.0.0"]  # 예시

        # Major 버전 증가
        max_major = max([int(v.split('.')[0]) for v in existing_versions])
        # max_major = 2
        new_version = f"{max_major + 1}.0.0"
        # new_version = "3.0.0"

    # 2. operation_type = 'update' (수정)
    elif operation_type == 'update':
        current_version = "1.0.0"  # 수정 대상 버전

        # 동일 Major 버전 중 Minor 최대값 찾기
        same_major_versions = ["1.0.0", "1.1.0", "1.2.0"]
        max_minor = max([int(v.split('.')[1]) for v in same_major_versions])
        # max_minor = 2

        # Minor 버전 증가
        current_major = int(current_version.split('.')[0])
        new_version = f"{current_major}.{max_minor + 1}.0"
        # new_version = "1.3.0"
```

**scope 파라미터**:
```python
# scope = 'module' (기본값)
# → 특정 ProjectModule 내에서만 버전 조회
existing_versions = AgentContent.query.filter(
    AgentContent.project_module_id == project_module_id,
    AgentContent.efct_fns_dt > datetime.utcnow()
).all()

# scope = 'project' (수정 시 사용)
# → 프로젝트 전체에서 버전 조회 (마이너 업데이트용)
existing_versions = AgentContent.query.join(ProjectModule).filter(
    ProjectModule.project_id == project_id,
    ProjectModule.module_type == 'test_agent',
    AgentContent.efct_fns_dt > datetime.utcnow()
).all()
```

**처리 흐름**:
```
1. ProjectModule 생성 (신규) 또는 조회 (기존)
   └─ project_module_service.create_module()
       ├─ module_type='test_agent'
       ├─ status='waiting' (시나리오만 저장 시)
       └─ efct_fns_dt=9999-12-31 (활성)

2. 버전 결정
   └─ version_service.set_version(operation_type='create')
       ├─ 기존 버전: []
       ├─ 신규 생성 → Major 증가
       └─ 결과: "1.0.0"

3. AgentContent 생성
   └─ agent_content_service.create_agent_content()
       ├─ project_module_id: 위에서 생성한 ID
       ├─ stage_type='test_scenario_generation'
       ├─ version='1.0.0'
       ├─ status='completed'
       ├─ output_json={'scenarios': [...]}
       └─ efct_fns_dt=9999-12-31 (활성)

4. AgentTask 생성 (이력 관리)
   └─ agent_task_service.create_agent_task()
       ├─ agent_content_id: 위에서 생성한 ID
       ├─ task_name='scenario_save'
       └─ status='completed'

5. DB 커밋
   └─ db.session.commit()
```

**코드 위치**: `test_scenario_service.py:415-563`

---

**버전 관리 상태별 플로우 (State Diagram)**:

```
┌──────────────────────────────────────────────────────────────────┐
│  상태 1: 최초 시나리오 생성 (v1.0.0)                              │
└──────────────────────────────────────────────────────────────────┘

[Generate] → LLM 생성 (is_saved=false)
    ↓
[Save] → DB 저장
    ├─ ProjectModule 생성
    │  ├─ version = "1.0.0"
    │  ├─ status = "waiting"  ⭐ 시나리오만 있음, 케이스 대기
    │  └─ efct_fns_dt = 9999-12-31 (활성)
    │
    └─ AgentContent 생성
       ├─ stage_type = "test_scenario_generation"
       ├─ version = "1.0.0"
       └─ output_json = {"scenarios": [...]}

┌──────────────────────────────────────────────────────────────────┐
│  상태 2: 케이스 추가 (v1.0.0 완성)                                │
└──────────────────────────────────────────────────────────────────┘

[Generate Case] → LLM 생성 (시나리오 v1.0.0 기반)
    ↓
[Save Case] → DB 저장
    ├─ 기존 ProjectModule 사용 (v1.0.0)
    │  ├─ status = "waiting" → "completed" ⭐ 사이클 완료
    │  └─ progress = 100
    │
    └─ AgentContent 생성
       ├─ stage_type = "test_case_generation"
       ├─ version = "1.0.0"  ⭐ 시나리오와 동일 버전
       └─ output_json = {"test_cases": [...]}

DB 상태:
┌─────────────────────────────────────────────────────────────────┐
│ ProjectModule (v1.0.0)                                          │
│   ├─ status = "completed"                                       │
│   ├─ AgentContent (시나리오, v1.0.0)                            │
│   └─ AgentContent (케이스, v1.0.0)                              │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  상태 3: 시나리오 수정 (v1.0.0 → v1.1.0)                          │
└──────────────────────────────────────────────────────────────────┘

[Update Scenario] → 시나리오만 수정
    ↓
버전 관리 프로세스:
    ├─ 1. 마이너 버전 증가
    │  └─ version_service.set_version(
    │      operation_type='update',
    │      scope='project'  ⭐ 프로젝트 전체 조회
    │  )
    │  └─ 1.0.0 → 1.1.0
    │
    ├─ 2. 새 ProjectModule 생성 (v1.1.0)
    │  ├─ version = "1.1.0"
    │  ├─ status = "completed"  ⭐ 기존 케이스도 복사됨
    │  └─ efct_fns_dt = 9999-12-31 (활성)
    │
    ├─ 3. 기존 ProjectModule 종료 (v1.0.0)
    │  └─ efct_fns_dt = now() (비활성)
    │
    ├─ 4. 새 AgentContent 생성 (수정된 시나리오, v1.1.0)
    │  ├─ stage_type = "test_scenario_generation"
    │  ├─ version = "1.1.0"
    │  └─ output_json = {"scenarios": [...]}  ⭐ 수정된 내용
    │
    ├─ 5. 기존 AgentContent 종료 (시나리오, v1.0.0)
    │  └─ efct_fns_dt = now() (비활성)
    │
    └─ 6. 기존 케이스 AgentContent 복사 (v1.0.0 → v1.1.0)
       └─ _copy_related_agent_contents()
          ├─ stage_type = "test_case_generation"
          ├─ version = "1.1.0"  ⭐ 새 버전으로 복사
          └─ output_json = {...}  ⭐ 내용은 동일

DB 상태 (수정 후):
┌─────────────────────────────────────────────────────────────────┐
│ ProjectModule (v1.0.0) - 비활성 (efct_fns_dt = now)             │
│   ├─ AgentContent (시나리오, v1.0.0) - 비활성                   │
│   └─ AgentContent (케이스, v1.0.0) - 비활성                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ ProjectModule (v1.1.0) - 활성 ⭐                                │
│   ├─ AgentContent (시나리오, v1.1.0) - 수정된 내용              │
│   └─ AgentContent (케이스, v1.1.0) - 복사된 내용                │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  상태 4: 신규 사이클 시작 (v2.0.0)                                │
└──────────────────────────────────────────────────────────────────┘

[Generate New Scenario] → 완전히 새로운 시나리오 생성
    ↓
[Save] → DB 저장
    └─ version_service.set_version(operation_type='create')
       └─ Major 버전 증가: 1.1.0 → 2.0.0

DB 상태:
┌─────────────────────────────────────────────────────────────────┐
│ ProjectModule (v1.0.0) - 비활성                                 │
│ ProjectModule (v1.1.0) - 비활성                                 │
│ ProjectModule (v2.0.0) - 활성 ⭐ 새 사이클                      │
│   └─ AgentContent (시나리오, v2.0.0)                            │
└─────────────────────────────────────────────────────────────────┘
```

**efct_fns_dt 필드 설명**:
```python
# efct_fns_dt = Effective Finish Date (유효 종료일)

# 활성 상태 (현재 사용 중)
efct_fns_dt = datetime(9999, 12, 31, 23, 59, 59)

# 비활성 상태 (이력 데이터)
efct_fns_dt = datetime.utcnow()  # 현재 시각

# 조회 시 활성 데이터만 가져오기
ProjectModule.query.filter(
    efct_fns_dt > datetime.utcnow()  # 9999-12-31 > now → True (활성)
).all()
```

---

#### 1.3 시나리오 조회 (List)
```http
GET /test-agent/scenario?project_id={id}&version={version}&page={page}
```

**쿼리 파라미터**:
- `project_id` (필수): 프로젝트 ID
- `version` (선택, 기본값: 최신 버전): 버전
- `page` (선택, 기본값: 1): 페이지 번호
- `page_size` (선택, 기본값: 20): 페이지 크기

**응답 예시**:
```json
{
  "scenarios": [...],
  "total_count": 15,
  "page": 1,
  "page_size": 20,
  "version": "1.0.0"
}
```

**DB 쿼리**:
```python
# test_scenario_service.py:711
ProjectModule.query.filter(
    ProjectModule.project_id == project_id,
    ProjectModule.module_type == 'test_agent',
    ProjectModule.version == version,
    ProjectModule.efct_fns_dt > datetime.utcnow()  # 활성 상태만
).all()
```

---

#### 1.4 시나리오 수정 (Update)
```http
POST /test-agent/scenario/update
Content-Type: application/json
```

**요청 파라미터**:
```json
{
  "agent_content_id": "시나리오 AgentContent ID",
  "version": "1.0.0",
  "scenarios": [...]  // 수정된 전체 시나리오 배열
}
```

**처리 흐름**:
```
1. 마이너 버전 증가
   └─ version_service.set_version(operation_type='update', scope='project')
   └─ 1.0.0 → 1.1.0
2. 새 ProjectModule 생성 (v1.1.0)
3. 기존 ProjectModule 종료 (efct_fns_dt = now)
4. 새 AgentContent 생성 (v1.1.0)
5. 기존 AgentContent 종료
6. 다른 단계 AgentContent 복사 (케이스도 함께 버전업)
```

**코드 위치**: `test_scenario_service.py:917-1007`

---

#### 1.5 시나리오 검색 (Search)
```http
GET /test-agent/scenario/search?project_id={id}&priorities=high,medium&name_query=로그인
```

**쿼리 파라미터**:
- `project_id` (필수): 프로젝트 ID
- `version` (선택): 버전
- `categories` (선택): 카테고리 배열 (예: "AUTH,USER")
- `priorities` (선택): 우선순위 배열 (예: "high,medium")
- `name_query` (선택): 시나리오명 검색어
- `page`, `page_size` (선택): 페이지네이션

**필터링 로직**:
```python
# test_scenario_service.py:774-792
def _match_category(item):
    c = item.get('category', {}).get('category', '').lower()
    ab = item.get('category', {}).get('category_abbreviation', '').lower()
    # 부분 매치: 카테고리명 또는 약어에 검색어 포함 시 매치
    for search_term in cate_set:
        if search_term in c or search_term in ab:
            return True
    return False
```

---

#### 1.6 시나리오 CSV 다운로드
```http
GET /test-agent/scenario/csv?project_id={id}&version={version}
```

**응답**: CSV 파일 다운로드
```csv
순번,시나리오 ID,카테고리,시나리오명,설명,우선순위,테스트 유형
1,TS_001,[AUTH] 인증/인가,사용자 로그인,ID/PW로 로그인,high,functional
```

**코드 위치**: `test_export_controller.py:23-88`

---

#### 1.7 시나리오 파일 임포트
```http
POST /test-agent/scenario/import
Content-Type: multipart/form-data
```

**요청 파라미터**:
- `repo_ids`: 저장소 파일 ID 배열 (CSV/Excel 파일)

**처리 흐름**:
```
1. repo_ids로 UploadFile 조회
2. pandas로 CSV/Excel 파싱
   └─ _parse_excel_to_scenarios()
3. 컬럼 별칭 매핑 (국문/영문 대응)
   └─ '시나리오 ID' / 'scenario_id' / 'scenarioId' 모두 인식
4. 시나리오 객체 구성
5. JSON 응답 (DB 저장 안함, 사용자 확인용)
```

**코드 위치**: `test_scenario_service.py:1009-1250`

---

### 2. 테스트 케이스 API

#### 2.1 케이스 생성 (Generate)
```http
POST /test-agent/case/generate
Content-Type: multipart/form-data
```

**요청 파라미터**:
```json
{
  "project_id": "프로젝트 ID (필수)",
  "project_module_id": "프로젝트 모듈 ID (필수)",
  "user_input": "사용자 입력 텍스트 (선택)",
  "scenarios": [...],  // 직접 전달된 시나리오 (선택)
  "test_scenario_agent_result_id": "시나리오 AgentContent ID (선택)",
  "files": "참조 파일 업로드 (선택)",
  "repo_ids": ["저장소 파일 ID 배열 (선택)"],
  "repo_keys": ["저장소 파일 키 배열 (선택)"]
}
```

**시나리오 소스 우선순위**:
```
1순위: scenarios (직접 전달)
2순위: test_scenario_agent_result_id (DB 조회)
3순위: project_module_id로 최신 시나리오 자동 조회
```

**응답 예시**:
```json
{
  "test_cases": [
    {
      "sno": 1,
      "testcase_id": "TC_001",
      "scenario_id": "TS_001",
      "scenario_name": "사용자 로그인 시나리오",
      "name": "정상 로그인 테스트",
      "category": {
        "category": "인증/인가",
        "category_abbreviation": "AUTH"
      },
      "priority": "high",
      "steps": [
        {
          "seq": "1",
          "action": "로그인 페이지 접속",
          "condition": "사용자가 로그아웃 상태",
          "input": "유효한 ID/PW",
          "expected": "로그인 성공, 메인 페이지 이동"
        }
      ],
      "expected_result": "로그인 성공 후 메인 페이지로 이동",
      "test_data": { "userId": "test@example.com", "password": "****" }
    }
  ],
  "total_count": 45,
  "is_saved": false,
  "can_save": true,
  "input_data": { ... }
}
```

**처리 흐름**:
```
1. 시나리오 소스 결정
   ├─ 직접 전달 (scenarios)
   ├─ DB 조회 (test_scenario_agent_result_id)
   └─ 자동 조회 (project_module_id)
2. 참조 파일 변환
   └─ repo_ids + repo_keys + files → Markdown
3. LLM 워크플로우 실행
   └─ workflow.case_workflow()
       ├─ ThreadPoolExecutor (5 workers)
       └─ test_case_agent() - 병렬 케이스 생성
4. 시나리오 정보 주입
   └─ scenario_id, category, priority 강제 설정
5. 결과 반환 (미저장)
```

**코드 위치**: `test_case_controller.py:85-166`

---

#### 2.2 케이스 저장 (Save)
```http
POST /test-agent/case
Content-Type: application/json
```

**처리 흐름**:
```
1. 시나리오와 동일한 ProjectModule 사용 (사이클 완성)
2. 시나리오 버전 조회
   └─ latest_scenario_content.version
3. AgentContent 생성 (동일 버전)
   └─ stage_type='test_case_generation'
4. ProjectModule 상태 변경
   └─ status='completed', progress=100
   └─ 시나리오+케이스 모두 완성 시 사이클 완료
```

**코드 위치**: `test_case_service.py:450-596`

---

#### 2.3 케이스 조회/검색/수정/임포트
시나리오 API와 동일한 구조로 제공됩니다.

**특이사항**:
- **search**: 시나리오 ID 필터 추가 (`scenario_ids` 파라미터)
- **import**: 스텝 데이터 그룹핑 (동일 케이스ID 행 → steps 배열)

---

### 3. 공통 API

#### 3.1 최신 버전 조회
```http
GET /test-agent/latest-version?project_id={id}&project_module_id={module_id}
```

**응답**:
```json
{
  "scenario_version": "1.2.0",
  "case_version": "1.2.0",
  "plan_version": null
}
```

---

#### 3.2 AgentContent 조회
```http
GET /test-agent/agent?module=scenario&project_id={id}&version={version}
```

**응답**: AgentContent 객체 (input_json, output_json 포함)

---

## 💼 핵심 비즈니스 로직

### 1. 통합 입력 처리 (Unified Input Processing)

**목적**: 다양한 입력 소스를 하나의 파이프라인으로 통합

**지원 입력**:
```python
# test_scenario_service.py:187-268
# 1. user_input (직접 텍스트)
user_input = scenario_input.user_input or ""

# 2. requirement_agent 결과 (요구사항 분석 결과)
requirement_result = self.file_conversion_service.get_requirement_agent_result(
    scenario_input.requirement_agent_result_id
)

# 3. 파일 (repo_ids + repo_keys + 직접 업로드)
file_document = self.file_conversion_service.convert_files_by_repo_keys_and_ids(
    repo_ids=scenario_input.repo_ids,
    repo_keys=scenario_input.repo_keys,
    files=scenario_input.files
)

# 모든 입력 통합
combined_input = user_input + "\n\n" + requirement_result + "\n\n" + file_document
```

**자동 폴백 메커니즘**:
```python
# test_case_service.py:218-228
if case_input.test_scenario_agent_result_id:
    # 1순위: 직접 전달된 ID
    scenario_result = get_test_scenario_agent_result(id)
else:
    # 2순위: project_module_id로 자동 조회
    scenario_result = get_test_scenario_agent_result_by_module_id(
        case_input.project_module_id
    )
```

---

### 2. 2단계 생성 프로세스 (Generate → Save)

**설계 의도**: 사용자가 LLM 결과를 확인 후 저장 여부 결정

```
┌─────────────────────────────────────────────────────────┐
│  1단계: Generate (생성)                                  │
│  ├─ LLM 호출 → 시나리오/케이스 생성                      │
│  ├─ is_saved = false                                    │
│  ├─ can_save = true                                     │
│  └─ input_data 포함 (재생성/저장 시 사용)                │
└─────────────────────────────────────────────────────────┘
                        ↓
         [사용자 확인 및 수정 가능]
                        ↓
┌─────────────────────────────────────────────────────────┐
│  2단계: Save (저장)                                      │
│  ├─ scenario_data/case_data 전달                        │
│  ├─ ProjectModule + AgentContent 생성                   │
│  ├─ is_saved = true                                     │
│  └─ can_proceed = true (다음 단계 진행 가능)             │
└─────────────────────────────────────────────────────────┘
```

**장점**:
- ✅ LLM 생성 실패 시 DB 오염 방지
- ✅ 사용자가 결과 확인 후 저장 결정
- ✅ DB 트랜잭션 최소화

---

### 3. 버전 관리 전략

**시맨틱 버저닝**: `Major.Minor.Patch`

```python
# version_service.py:70-196
def set_version(operation_type, scope):
    if operation_type == 'create':
        # 신규 생성: Major 증가
        # 1.0.0 → 2.0.0
        max_major = max([int(v.split('.')[0]) for v in existing_versions])
        return f"{max_major + 1}.0.0"

    elif operation_type == 'update':
        # 수정: Minor 증가
        # 1.0.0 → 1.1.0
        current_major = int(current_version.split('.')[0])
        max_minor = max([int(v.split('.')[1]) for v in same_major_versions])
        return f"{current_major}.{max_minor + 1}.0"
```

**scope 파라미터**:
- `module`: 특정 ProjectModule 내에서만 버전 조회
- `project`: 프로젝트 전체에서 버전 조회 (마이너 업데이트 시 사용)

**사이클 관리**:
```
시나리오 생성 (v1.0.0)
    ↓
케이스 생성 (v1.0.0) → 사이클 완료 (ProjectModule.status = 'completed')

시나리오 수정 (v1.1.0)
    ↓
케이스 자동 복사 (v1.1.0) → 새 사이클 시작
```

---

### 4. LLM 워크플로우 (병렬 처리)

**시나리오 생성 워크플로우**:
```python
# workflow.py:353-469
def scenario_workflow(requirement):
    # Step 1: 플래너 에이전트 (카테고리 분류)
    result = planner_agent(requirement)
    test_scenario_data = test_scenario_json_parser(result)
    # → ["인증/인가", "사용자 관리", "결제", ...]

    # Step 2: 병렬 시나리오 생성 (5 workers)
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = {
            executor.submit(process_category, category, i): category
            for i, category in enumerate(test_scenario_data)
        }

        for future in as_completed(futures):
            scenario_data = future.result()
            scenarios.extend(scenario_data)

    return scenarios
```

**케이스 생성 워크플로우**:
```python
# workflow.py:472-630
def case_workflow(scenarios, reference_doc):
    # 병렬 케이스 생성 (5 workers)
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = {
            executor.submit(process_scenario, scenario, i): scenario
            for i, scenario in enumerate(scenarios)
        }

        for future in as_completed(futures):
            result = future.result()
            all_test_cases.extend(result['test_cases'])

    return {'test_cases': all_test_cases, 'total_count': len(all_test_cases)}
```

**재시도 메커니즘**:
```python
# workflow.py:496-574
max_retries = 3
for retry in range(max_retries):
    try:
        result = test_case_agent(template)
        cases = test_case_json_parser(result)
        if cases:
            break  # 성공
    except Exception as e:
        if retry < max_retries - 1:
            wait_time = 2 ** retry  # 지수 백오프: 1초, 2초, 4초
            time.sleep(wait_time)
```

---

### 5. 파일 변환 시스템

**지원 포맷**:
```yaml
문서: PDF, Word (docx), Markdown, TXT
스프레드시트: Excel (xlsx, xls), CSV
이미지: PNG, JPG (OCR 미지원)
```

**변환 파이프라인**:
```python
# file_conversion_service.py:459-552
def convert_bytes_to_markdown(file_data, filename):
    # 1차 시도: convert_stream (in-memory)
    try:
        bytes_stream = io.BytesIO(file_data)
        result = self.markitdown.convert_stream(
            bytes_stream,
            file_extension=file_ext
        )
        return result.text_content
    except:
        # 2차 시도: convert (temp file)
        with tempfile.NamedTemporaryFile(suffix=file_ext) as temp_file:
            temp_file.write(file_data)
            result = self.markitdown.convert(temp_file.name)
            return result.text_content
```

**repo_ids vs repo_keys**:
```python
# repo_ids: DB의 UploadFile ID → 메타데이터 포함
# repo_keys: Storage 직접 키 → 빠른 접근
```

---

## 📊 데이터 플로우

### 시나리오 생성 플로우
```
[사용자 요청]
    ↓
POST /test-agent/scenario/generate
    ↓
TestScenarioController.generate()
    ├─ 입력 검증 (TestScenarioInput)
    └─ TestScenarioService.create_test_scenarios_with_unified_inputs()
        ├─ 1. 입력 통합
        │   ├─ user_input
        │   ├─ requirement_agent_result (DB 조회)
        │   └─ files (repo_ids + repo_keys)
        │       └─ FileConversionService.convert_files_by_repo_keys_and_ids()
        │           ├─ UploadFile 조회
        │           ├─ storage.load(key)
        │           └─ MarkItDown.convert_stream()
        │
        ├─ 2. LLM 워크플로우 실행
        │   └─ workflow.scenario_workflow()
        │       ├─ planner_agent() → 카테고리 분류
        │       └─ ThreadPoolExecutor (5 workers)
        │           └─ scenario_agent() → 시나리오 생성
        │
        └─ 3. 결과 반환 (미저장)
            └─ {"scenarios": [...], "is_saved": false, "can_save": true}

[사용자 확인/수정]
    ↓
POST /test-agent/scenario (저장)
    ↓
TestScenarioService.save_scenario_data()
    ├─ ProjectModuleService.create_module()
    ├─ VersionService.set_version() → "1.0.0"
    ├─ AgentContentService.create_agent_content()
    │   └─ stage_type='test_scenario_generation'
    │   └─ output_json={'scenarios': [...]}
    ├─ AgentTaskService.create_agent_task()
    └─ DB.commit()
        └─ {"is_saved": true, "can_proceed": true}
```

### 케이스 생성 플로우
```
POST /test-agent/case/generate
    ↓
TestCaseService.create_test_cases_with_unified_inputs()
    ├─ 1. 시나리오 소스 결정
    │   ├─ 직접 전달 (scenarios)
    │   ├─ DB 조회 (test_scenario_agent_result_id)
    │   │   └─ AgentContent.query.filter_by(id=...).first()
    │   │   └─ output_json['scenarios'] 추출
    │   └─ 자동 조회 (project_module_id)
    │       └─ stage_type='test_scenario_generation' 최신 조회
    │
    ├─ 2. 참조 파일 변환
    │   └─ repo_ids + repo_keys + files → Markdown
    │
    ├─ 3. LLM 워크플로우 실행
    │   └─ workflow.case_workflow(scenarios, reference_doc)
    │       └─ ThreadPoolExecutor (5 workers)
    │           └─ test_case_agent() → 케이스 생성
    │
    ├─ 4. 시나리오 정보 주입
    │   └─ 각 케이스에 scenario_id, category, priority 강제 설정
    │
    └─ 5. 결과 반환 (미저장)
        └─ {"test_cases": [...], "is_saved": false, "can_save": true}

POST /test-agent/case (저장)
    ↓
TestCaseService.save_case_data()
    ├─ 시나리오와 동일한 ProjectModule 사용
    ├─ 시나리오 버전 조회 (동일 버전 유지)
    ├─ AgentContentService.create_agent_content()
    │   └─ stage_type='test_case_generation'
    ├─ ProjectModuleService.update_module()
    │   └─ status='completed' (사이클 완료)
    └─ DB.commit()
```

### 수정 플로우 (버전 증가)
```
POST /test-agent/scenario/update
    ↓
TestScenarioService.update_test_scenarios_full_replace()
    ├─ 1. 마이너 버전 증가
    │   └─ VersionService.set_version(operation_type='update')
    │       └─ 1.0.0 → 1.1.0
    │
    ├─ 2. 새 ProjectModule 생성 (v1.1.0)
    │   └─ ProjectModuleService.create_module()
    │
    ├─ 3. 기존 ProjectModule 종료
    │   └─ efct_fns_dt = datetime.utcnow()
    │
    ├─ 4. 새 AgentContent 생성 (v1.1.0)
    │   └─ stage_type='test_scenario_generation'
    │
    ├─ 5. 기존 AgentContent 종료
    │   └─ efct_fns_dt = datetime.utcnow()
    │
    ├─ 6. 다른 단계 AgentContent 복사 (케이스도 v1.1.0으로)
    │   └─ _copy_related_agent_contents()
    │
    └─ DB.commit()
```

---

## 📈 코드 품질 분석

### 1. 강점 (Strengths)

#### ✅ 명확한 계층 분리
```python
# Controller는 요청/응답만
class TestScenarioApi(Resource):
    @jwt_required()
    def post(self):
        result = self.scenario_service.create_test_scenarios(...)
        return result, 200

# Service는 비즈니스 로직만
class TestScenarioService:
    def create_test_scenarios(...):
        # 파일 변환, LLM 호출, DB 저장

# Agent는 LLM 호출만
class TestScenario(BaseAgent):
    def chat(self, query, user):
        return self.client.send_message(...)
```

#### ✅ Pydantic 기반 타입 안전성
```python
# model_dto/test_agent_dto.py
class TestScenarioInput(BaseModel):
    project_id: str = Field(description="프로젝트 ID")
    user_input: Optional[str] = Field(default="", description="사용자 입력")
    files: Optional[List[Union[FileStorage, str, FileInfo]]] = Field(default=[])
    # ... 타입 검증 자동화
```

#### ✅ 상세한 로깅
```python
logger.info(f"📁 참조 파일 변환 시작 - repo_ids: {repo_ids}")
logger.info(f"✅ 파일 변환 완료: {len(file_document)}자")
logger.error(f"❌ 파일 로드 실패: {repo_key}, 에러: {str(e)}")
```

#### ✅ 병렬 처리 최적화
```python
# 5개 스레드로 시나리오/케이스 병렬 생성
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = {...}
    for future in as_completed(futures):
        results.extend(future.result())
```

#### ✅ 재시도 메커니즘
```python
max_retries = 3
for retry in range(max_retries):
    try:
        result = test_case_agent(...)
        break
    except Exception as e:
        wait_time = 2 ** retry  # 지수 백오프
        time.sleep(wait_time)
```

---

### 2. 개선 필요 사항 (Areas for Improvement)

#### ⚠️ God Class 문제 (1,400+ 라인)
```python
# test_scenario_service.py: 1,347 lines
# test_case_service.py: 1,417 lines

# 문제점:
# - 생성, 저장, 조회, 수정, 삭제, 검색, 파일 처리 모두 포함
# - 유지보수 어려움
# - 테스트 작성 어려움
```

**개선안**:
```python
# 책임 분리
class TestScenarioGenerationService:
    """LLM 생성만 담당"""
    def generate_scenarios(self, input_data) -> Dict

class TestScenarioPersistenceService:
    """DB 저장/조회만 담당"""
    def save_scenarios(self, scenario_data) -> Dict
    def get_scenarios(self, project_id, version) -> List

class TestScenarioSearchService:
    """검색/필터링만 담당"""
    def search_scenarios(self, filters) -> List

class TestScenarioFileService:
    """파일 처리만 담당"""
    def import_from_excel(self, file) -> List
    def export_to_csv(self, scenarios) -> str
```

#### ⚠️ 중복 코드
**test_case_service.py:287-333 vs test_scenario_service.py:248-289**
```python
# AgentContent.output_json 추출 로직 중복
agent_content = AgentContent.query.filter_by(id=id).first()
if agent_content.output_json and 'scenarios' in agent_content.output_json:
    scenarios = agent_content.output_json['scenarios']
    # ... 동일한 필드 보장 로직
```

**개선안**:
```python
# utils/agent_content_utils.py
class AgentContentExtractor:
    @staticmethod
    def extract_scenarios(agent_content_id: str) -> List[Dict]:
        agent_content = AgentContent.query.filter_by(id=agent_content_id).first()
        if not agent_content or not agent_content.output_json:
            return []

        scenarios = agent_content.output_json.get('scenarios', [])

        # 필드 보장
        for scenario in scenarios:
            if 'name' not in scenario and 'scenario_name' in scenario:
                scenario['name'] = scenario['scenario_name']

        return scenarios

    @staticmethod
    def extract_test_cases(agent_content_id: str) -> List[Dict]:
        # 케이스 추출 로직
        pass
```

#### ⚠️ 하드코딩된 값
```python
# workflow.py:311, 426, 577
max_workers=5  # 하드코딩

# test_case_service.py:497
max_retries = 3  # 하드코딩

# workflow.py:562
wait_time = 2 ** retry  # Magic number
```

**개선안**:
```python
# config.py 또는 .env
TEST_AGENT_MAX_WORKERS=5
TEST_AGENT_MAX_RETRIES=3
TEST_AGENT_RETRY_BACKOFF_FACTOR=2

# workflow.py
from config import get_env

MAX_WORKERS = int(get_env('TEST_AGENT_MAX_WORKERS') or '5')
MAX_RETRIES = int(get_env('TEST_AGENT_MAX_RETRIES') or '3')
RETRY_BACKOFF = int(get_env('TEST_AGENT_RETRY_BACKOFF_FACTOR') or '2')
```

#### ⚠️ 예외 처리 불일치
```python
# 패턴 1: ValueError raise
def create_test_cases(...):
    try:
        ...
    except Exception as e:
        raise ValueError(f"테스트 케이스 생성 실패: {str(e)}")

# 패턴 2: error dict 반환
def create_test_cases(...):
    try:
        ...
    except Exception as e:
        return {
            "test_cases": [],
            "error": str(e),
            "is_saved": False
        }
```

**개선안**:
```python
# exceptions.py
class TestAgentError(Exception):
    def __init__(self, message: str, code: str = 'UNKNOWN', details: Dict = None):
        self.message = message
        self.code = code
        self.details = details or {}
        super().__init__(message)

# 통일된 예외 처리
try:
    result = service.create_test_cases(...)
except TestAgentError as e:
    return {
        "error": {
            "message": e.message,
            "code": e.code,
            "details": e.details
        }
    }, 400
```

#### ⚠️ DB 쿼리 최적화 (N+1 문제)
```python
# test_case_service.py:790-806
for project_module in project_modules:  # N개
    agent_contents = AgentContent.query.filter(
        AgentContent.project_module_id == project_module.id,  # +1 쿼리
        # ...
    ).all()
```

**개선안**:
```python
# join + subquery로 1번의 쿼리로 해결
from sqlalchemy.orm import joinedload

result = db.session.query(AgentContent).options(
    joinedload(AgentContent.project_module)
).join(
    ProjectModule
).filter(
    ProjectModule.project_id == project_id,
    ProjectModule.module_type == 'test_agent',
    AgentContent.version == version,
    ProjectModule.efct_fns_dt > datetime.utcnow(),
    AgentContent.efct_fns_dt > datetime.utcnow()
).all()
```

#### ⚠️ print 문 남아있음 (디버깅 코드)
```python
# version_service.py:84-188
print(f"\n🚀 test_agent VersionService.set_version() 시작")
print(f"📋 입력 파라미터:")
print(f"   └─ project_id: {project_id}")
# ... 총 30+ print 문
```

**개선안**:
```python
logger.debug(f"VersionService.set_version() 시작")
logger.debug(f"입력: project_id={project_id}, operation_type={operation_type}")
```

#### ⚠️ 긴 메서드 (200+ 라인)
```python
# test_case_service.py:175-448 (273 lines)
def create_test_cases_with_unified_inputs(...):
    # 파일 분류 (30 lines)
    # 파일 변환 (40 lines)
    # 시나리오 처리 (80 lines)
    # 워크플로우 실행 (50 lines)
    # 예외 처리 (40 lines)
```

**개선안**:
```python
def create_test_cases_with_unified_inputs(self, case_input, current_user):
    """통합 입력 처리 (5단계 파이프라인)"""
    # 1. 파일 분류
    classified_files = self._classify_files(case_input.files)

    # 2. 파일 변환
    reference_doc = self._convert_reference_files(
        classified_files.reference,
        case_input.repo_ids,
        case_input.repo_keys
    )

    # 3. 시나리오 해결
    scenarios = self._resolve_scenarios(
        case_input,
        classified_files.scenario
    )

    # 4. 워크플로우 실행
    result = self._execute_case_workflow(
        scenarios,
        reference_doc,
        case_input
    )

    # 5. 응답 포맷팅
    return self._format_generate_response(result, case_input)

def _classify_files(self, files) -> ClassifiedFiles:
    """파일 분류 (시나리오 vs 참조)"""
    pass

def _convert_reference_files(self, files, repo_ids, repo_keys) -> str:
    """참조 파일 변환"""
    pass

def _resolve_scenarios(self, case_input, scenario_files) -> List[Dict]:
    """시나리오 소스 결정 (3가지 우선순위)"""
    pass
```

#### ⚠️ 타입 힌트 불완전
```python
# test_scenario_service.py:189
def create_test_scenarios_with_unified_inputs(self, scenario_input, current_user):
    # ❌ 파라미터 타입 없음
```

**개선안**:
```python
from typing import Dict, Any
from model_dto.test_agent_dto import TestScenarioInput
from models.account import User

def create_test_scenarios_with_unified_inputs(
    self,
    scenario_input: TestScenarioInput,
    current_user: User
) -> Dict[str, Any]:
    """통합 입력 처리 방식으로 테스트 시나리오 생성

    Args:
        scenario_input: 시나리오 입력 DTO
        current_user: 현재 사용자

    Returns:
        시나리오 생성 결과 딕셔너리
        - scenarios: 생성된 시나리오 목록
        - is_saved: DB 저장 여부
        - can_save: 저장 가능 여부

    Raises:
        TestAgentError: 시나리오 생성 실패 시
    """
    pass
```

---

## 🚀 개선 권장사항

### 우선순위 1: 즉시 개선 (1주 이내)

#### 1. print → logger 전환
```python
# Before (version_service.py:84)
print(f"\n🚀 test_agent VersionService.set_version() 시작")

# After
logger.debug(f"VersionService.set_version() 시작")
```

#### 2. 하드코딩 제거
```python
# .env 추가
TEST_AGENT_MAX_WORKERS=5
TEST_AGENT_MAX_RETRIES=3
TEST_AGENT_RETRY_BACKOFF_FACTOR=2
TEST_AGENT_BASE_TIMEOUT=300

# workflow.py
MAX_WORKERS = int(get_env('TEST_AGENT_MAX_WORKERS') or '5')
```

#### 3. 예외 처리 통일
```python
# exceptions.py 생성
class TestAgentError(Exception):
    pass

# 모든 서비스에서 통일된 처리
```

---

### 우선순위 2: 단기 개선 (2-4주)

#### 4. 서비스 클래스 분리
```python
# 현재: TestScenarioService (1,347 lines)
# 개선: 4개 서비스로 분리
TestScenarioGenerationService  # 생성
TestScenarioPersistenceService # 저장/조회
TestScenarioSearchService      # 검색
TestScenarioFileService        # 파일 처리
```

#### 5. 중복 코드 제거
```python
# utils/agent_content_utils.py
class AgentContentExtractor:
    @staticmethod
    def extract_scenarios(agent_content_id) -> List[Dict]

    @staticmethod
    def extract_test_cases(agent_content_id) -> List[Dict]
```

#### 6. DB 쿼리 최적화
```python
# N+1 문제 해결
# - joinedload 사용
# - subquery 활용
# - 인덱스 추가 검토
```

#### 7. 타입 힌트 추가
```python
# 모든 public 메서드에 타입 힌트 추가
def create_test_scenarios(
    self,
    scenario_input: TestScenarioInput,
    current_user: User
) -> Dict[str, Any]:
```

---

### 우선순위 3: 장기 개선 (1-3개월)

#### 8. 테스트 코드 작성
```python
# tests/services/test_scenario_service_test.py
class TestScenarioServiceTest:
    def test_create_scenarios_with_user_input(self):
        # Given
        scenario_input = TestScenarioInput(...)

        # When
        result = service.create_test_scenarios(...)

        # Then
        assert result['is_saved'] == False
        assert len(result['scenarios']) > 0

    def test_save_scenarios(self):
        # Given
        scenario_data = {...}

        # When
        result = service.save_scenario_data(...)

        # Then
        assert result['is_saved'] == True
        assert result['version'] == '1.0.0'
```

**목표 커버리지**: 70% 이상

#### 9. 긴 메서드 리팩토링
```python
# 200+ 라인 메서드를 5개 이하 서브 메서드로 분리
# 각 메서드는 하나의 책임만 수행
```

#### 10. API 문서 자동화
```python
# Swagger/OpenAPI 통합
from flask_restx import Api, Resource, fields

api = Api(
    title='Test Agent API',
    version='2.0',
    description='AI 기반 테스트 자동화 API'
)

scenario_model = api.model('TestScenario', {
    'scenario_id': fields.String(required=True),
    'scenario_name': fields.String(required=True),
    # ...
})

@api.route('/test-agent/scenario/generate')
class TestScenarioGenerate(Resource):
    @api.doc('generate_scenarios')
    @api.expect(scenario_input_model)
    @api.marshal_with(scenario_response_model)
    def post(self):
        """테스트 시나리오 생성 (미저장)"""
        pass
```

---



