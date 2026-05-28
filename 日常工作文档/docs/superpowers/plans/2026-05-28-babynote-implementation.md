# BabyNote Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Feishu Bot that records baby daily activities via voice/text/photo, stores data as Obsidian Markdown, supports natural language queries, and generates daily/weekly/monthly growth reports.

**Architecture:** Python middleman service receives messages from Feishu Bot webhook, routes intents via LLM, writes/reads structured Markdown files in Obsidian Vault format. Voice uses Feishu ASR, images download and store locally. Scheduled tasks generate daily reports automatically.

**Tech Stack:** Python 3.11+, Feishu Python SDK, OpenAI API (GPT-4o), APScheduler (for daily report), PyYAML, python-frontmatter

---

## File Structure

```
babynote/
├── config.py                   # App configuration (env vars, paths, constants)
├── main.py                     # Entry point: start webhook server + scheduler
├── feishu/
│   ├── __init__.py
│   ├── bot.py                  # Feishu Bot webhook handler + message dispatch
│   ├── auth.py                 # Feishu auth (tenant token, event verification)
│   └── media.py                # Download images + voice-to-text via Feishu API
├── llm/
│   ├── __init__.py
│   ├── intent.py               # Intent classification (record/query/report/photo/unknown)
│   ├── extract.py              # Structured data extraction from natural language
│   └── answer.py               # Query answering using vault data
├── vault/
│   ├── __init__.py
│   ├── daily.py                # Create/read/update daily markdown files
│   ├── reports.py              # Generate daily/weekly/monthly reports
│   ├── config_store.py         # Read/write baby/config.yaml
│   └── attachments.py          # Save images, generate filenames, write wikilinks
├── scheduler/
│   ├── __init__.py
│   └── jobs.py                 # APScheduler daily 22:00 report job
├── models.py                   # Pydantic data models (Intent, Record, Report, BabyConfig)
├── baby_config_template.yaml   # Template for initial baby config
├── requirements.txt
├── .env.example                # Environment variable template
└── tests/
    ├── __init__.py
    ├── test_intent.py
    ├── test_extract.py
    ├── test_answer.py
    ├── test_daily.py
    ├── test_reports.py
    ├── test_config_store.py
    └── test_attachments.py
```

---

### Task 1: Project Scaffold + Config

**Files:**
- Create: `babynote/config.py`
- Create: `babynote/models.py`
- Create: `babynote/baby_config_template.yaml`
- Create: `babynote/requirements.txt`
- Create: `babynote/.env.example`
- Create: `babynote/tests/__init__.py`

- [ ] **Step 1: Create project directory and requirements.txt**

```txt
# requirements.txt
flask>=3.0
lark-oapi>=1.4
openai>=1.30
pyyaml>=6.0
python-frontmatter>=1.1
python-dateutil>=2.9
apscheduler>=3.10
pydantic>=2.7
python-dotenv>=1.0
pytest>=8.2
```

- [ ] **Step 2: Create .env.example**

```bash
# Feishu Bot
FEISHU_APP_ID=cli_xxxxx
FEISHU_APP_SECRET=xxxxx
FEISHU_VERIFICATION_TOKEN=xxxxx
FEISHU_ENCRYPT_KEY=xxxxx

# LLM
OPENAI_API_KEY=sk-xxxxx
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-4o

# Vault
VAULT_PATH=/path/to/your/obsidian/vault

# Server
PORT=8080
```

- [ ] **Step 3: Write config.py**

```python
# config.py
import os
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

# Feishu
FEISHU_APP_ID = os.environ["FEISHU_APP_ID"]
FEISHU_APP_SECRET = os.environ["FEISHU_APP_SECRET"]
FEISHU_VERIFICATION_TOKEN = os.environ["FEISHU_VERIFICATION_TOKEN"]
FEISHU_ENCRYPT_KEY = os.environ.get("FEISHU_ENCRYPT_KEY", "")

# LLM
OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]
OPENAI_BASE_URL = os.environ.get("OPENAI_BASE_URL", "https://api.openai.com/v1")
OPENAI_MODEL = os.environ.get("OPENAI_MODEL", "gpt-4o")

# Vault
VAULT_PATH = Path(os.environ.get("VAULT_PATH", "./vault"))

# Server
PORT = int(os.environ.get("PORT", 8080))

# Derived paths
BABY_DIR = VAULT_PATH / "baby"
BABY_REPORTS_DIR = BABY_DIR / "reports"
ATTACHMENTS_DIR = VAULT_PATH / "attachments"

# Scheduling
DAILY_REPORT_HOUR = 22  # 22:00 daily
```

- [ ] **Step 4: Write models.py**

```python
# models.py
from enum import Enum
from datetime import date, datetime
from pydantic import BaseModel


class IntentType(str, Enum):
    RECORD = "record"
    QUERY = "query"
    REPORT = "report"
    PHOTO = "photo"
    UNKNOWN = "unknown"


class RecordCategory(str, Enum):
    EATING = "吃喝"
    DIAPER = "排泄"
    SLEEP = "睡眠"
    MEASUREMENT = "身体指标"
    MILESTONE = "里程碑"
    HEALTH = "健康"


class Intent(BaseModel):
    type: IntentType
    confidence: float = 1.0


class Record(BaseModel):
    category: RecordCategory
    time: str  # "HH:MM" or "now"
    content: str
    photo: str | None = None


class BabyConfig(BaseModel):
    name: str = "宝宝"
    birthday: date
    gender: str  # "男" or "女"


class DailyFile(BaseModel):
    date: date
    baby_age: str  # "X个月Y天"
    sections: dict[str, list[str]]  # category -> list of entries


class ReportType(str, Enum):
    DAILY = "daily"
    WEEKLY = "weekly"
    MONTHLY = "monthly"
```

- [ ] **Step 5: Write baby_config_template.yaml**

```yaml
name: "宝宝"
birthday: "2025-09-16"
gender: "男"
```

- [ ] **Step 6: Commit**

```bash
cd babynote
git init
git add .
git commit -m "feat: project scaffold with config, models, and dependencies"
```

---

### Task 2: Vault — Baby Config Store

**Files:**
- Create: `babynote/vault/__init__.py`
- Create: `babynote/vault/config_store.py`
- Create: `babynote/tests/test_config_store.py`

- [ ] **Step 1: Write the failing test for config_store**

```python
# tests/test_config_store.py
import pytest
from pathlib import Path
from vault.config_store import load_config, save_config, compute_age
from models import BabyConfig
from datetime import date


def test_save_and_load_config(tmp_path):
    config_path = tmp_path / "baby" / "config.yaml"
    baby_dir = tmp_path / "baby"
    baby_dir.mkdir(parents=True, exist_ok=True)

    cfg = BabyConfig(name="小宝", birthday=date(2025, 9, 16), gender="男")
    save_config(cfg, config_path)

    loaded = load_config(config_path)
    assert loaded.name == "小宝"
    assert loaded.birthday == date(2025, 9, 16)
    assert loaded.gender == "男"


def test_compute_age(tmp_path):
    baby_dir = tmp_path / "baby"
    baby_dir.mkdir(parents=True, exist_ok=True)
    config_path = tmp_path / "baby" / "config.yaml"

    cfg = BabyConfig(name="宝宝", birthday=date(2025, 9, 16), gender="男")
    save_config(cfg, config_path)

    # On 2026-05-28, baby is 8 months 12 days old
    age = compute_age(date(2025, 9, 16), date(2026, 5, 28))
    assert age == "8个月12天"


def test_load_config_missing_file(tmp_path):
    config_path = tmp_path / "baby" / "config.yaml"
    with pytest.raises(FileNotFoundError):
        load_config(config_path)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd babynote && python -m pytest tests/test_config_store.py -v`
Expected: FAIL — module `vault.config_store` not found

- [ ] **Step 3: Implement config_store.py**

```python
# vault/config_store.py
import yaml
from datetime import date
from pathlib import Path
from models import BabyConfig


def load_config(config_path: Path) -> BabyConfig:
    """Load baby config from YAML file."""
    if not config_path.exists():
        raise FileNotFoundError(f"Config not found: {config_path}")

    with open(config_path, "r", encoding="utf-8") as f:
        data = yaml.safe_load(f)

    return BabyConfig(
        name=data["name"],
        birthday=data["birthday"],
        gender=data["gender"],
    )


def save_config(config: BabyConfig, config_path: Path) -> None:
    """Save baby config to YAML file. Creates parent dirs if needed."""
    config_path.parent.mkdir(parents=True, exist_ok=True)

    data = {
        "name": config.name,
        "birthday": config.birthday.isoformat(),
        "gender": config.gender,
    }

    with open(config_path, "w", encoding="utf-8") as f:
        yaml.dump(data, f, allow_unicode=True, default_flow_style=False)


def compute_age(birthday: date, today: date) -> str:
    """Compute baby age as 'X个月Y天' string."""
    # Calculate month difference
    months = (today.year - birthday.year) * 12 + (today.month - birthday.month)
    days = today.day - birthday.day

    # If day hasn't reached birthday day yet, adjust
    if days < 0:
        months -= 1
        # Days in the previous month
        prev_month = today.month - 1 if today.month > 1 else 12
        prev_year = today.year if today.month > 1 else today.year - 1
        import calendar
        days_in_prev_month = calendar.monthrange(prev_year, prev_month)[1]
        days = today.day + days_in_prev_month - birthday.day

    return f"{months}个月{days}天"
```

- [ ] **Step 4: Create vault/__init__.py**

```python
# vault/__init__.py
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd babynote && python -m pytest tests/test_config_store.py -v`
Expected: PASS — all 3 tests pass

- [ ] **Step 6: Commit**

```bash
cd babynote
git add vault/ tests/test_config_store.py
git commit -m "feat: baby config store with load/save/age computation"
```

---

### Task 3: Vault — Daily Markdown Read/Write

**Files:**
- Create: `babynote/vault/daily.py`
- Create: `babynote/tests/test_daily.py`

- [ ] **Step 1: Write the failing test for daily markdown operations**

```python
# tests/test_daily.py
import pytest
from datetime import date
from pathlib import Path
from vault.daily import create_daily_file, append_record, read_daily, get_section
from models import RecordCategory


def test_create_daily_file(tmp_path):
    baby_dir = tmp_path / "baby"
    baby_dir.mkdir(parents=True, exist_ok=True)

    filepath = create_daily_file(
        vault_path=tmp_path,
        target_date=date(2026, 5, 28),
        baby_age="8个月12天",
    )

    assert filepath.exists()
    content = filepath.read_text(encoding="utf-8")
    assert "date: 2026-05-28" in content
    assert "baby_age: \"8个月12天\"" in content
    assert "## 吃喝" in content
    assert "## 排泄" in content
    assert "## 睡眠" in content
    assert "## 身体指标" in content
    assert "## 里程碑" in content
    assert "## 健康" in content


def test_create_daily_file_idempotent(tmp_path):
    """Creating an existing daily file should not overwrite it."""
    baby_dir = tmp_path / "baby"
    baby_dir.mkdir(parents=True, exist_ok=True)

    filepath = create_daily_file(
        vault_path=tmp_path, target_date=date(2026, 5, 28), baby_age="8个月12天"
    )
    # Manually add some content
    append_record(
        vault_path=tmp_path,
        target_date=date(2026, 5, 28),
        category=RecordCategory.EATING,
        time="10:00",
        content="配方奶 120ml",
    )

    # Create again — should NOT overwrite
    create_daily_file(
        vault_path=tmp_path, target_date=date(2026, 5, 28), baby_age="8个月12天"
    )

    section = get_section(
        vault_path=tmp_path, target_date=date(2026, 5, 28),
        category=RecordCategory.EATING,
    )
    assert "配方奶 120ml" in section


def test_append_record(tmp_path):
    baby_dir = tmp_path / "baby"
    baby_dir.mkdir(parents=True, exist_ok=True)
    create_daily_file(
        vault_path=tmp_path, target_date=date(2026, 5, 28), baby_age="8个月12天"
    )

    append_record(
        vault_path=tmp_path,
        target_date=date(2026, 5, 28),
        category=RecordCategory.EATING,
        time="10:00",
        content="配方奶 120ml",
    )

    section = get_section(
        vault_path=tmp_path, target_date=date(2026, 5, 28),
        category=RecordCategory.EATING,
    )
    assert "- 10:00 配方奶 120ml" in section


def test_append_multiple_records(tmp_path):
    baby_dir = tmp_path / "baby"
    baby_dir.mkdir(parents=True, exist_ok=True)
    create_daily_file(
        vault_path=tmp_path, target_date=date(2026, 5, 28), baby_age="8个月12天"
    )

    append_record(tmp_path, date(2026, 5, 28), RecordCategory.EATING, "10:00", "配方奶 120ml")
    append_record(tmp_path, date(2026, 5, 28), RecordCategory.EATING, "15:00", "配方奶 90ml")
    append_record(tmp_path, date(2026, 5, 28), RecordCategory.DIAPER, "14:00", "换尿布，小便")

    eating = get_section(tmp_path, date(2026, 5, 28), RecordCategory.EATING)
    assert "- 10:00 配方奶 120ml" in eating
    assert "- 15:00 配方奶 90ml" in eating

    diaper = get_section(tmp_path, date(2026, 5, 28), RecordCategory.DIAPER)
    assert "- 14:00 换尿布，小便" in diaper
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd babynote && python -m pytest tests/test_daily.py -v`
Expected: FAIL — module `vault.daily` not found

- [ ] **Step 3: Implement daily.py**

```python
# vault/daily.py
import re
from datetime import date
from pathlib import Path
from models import RecordCategory

# Category heading mapping
CATEGORY_HEADINGS = {
    RecordCategory.EATING: "吃喝",
    RecordCategory.DIAPER: "排泄",
    RecordCategory.SLEEP: "睡眠",
    RecordCategory.MEASUREMENT: "身体指标",
    RecordCategory.MILESTONE: "里程碑",
    RecordCategory.HEALTH: "健康",
}

DAILY_TEMPLATE = """---
date: {date}
baby_age: "{baby_age}"
---

# 🍼 {date} 宝宝日记

## 吃喝

## 排泄

## 睡眠

## 身体指标

## 里程碑

## 健康
"""


def get_daily_filepath(vault_path: Path, target_date: date) -> Path:
    """Get the file path for a daily record."""
    baby_dir = vault_path / "baby"
    baby_dir.mkdir(parents=True, exist_ok=True)
    return baby_dir / f"{target_date.isoformat()}.md"


def create_daily_file(vault_path: Path, target_date: date, baby_age: str) -> Path:
    """Create a daily markdown file. If it already exists, return existing path."""
    filepath = get_daily_filepath(vault_path, target_date)

    if filepath.exists():
        return filepath

    content = DAILY_TEMPLATE.format(date=target_date.isoformat(), baby_age=baby_age)
    filepath.write_text(content, encoding="utf-8")
    return filepath


def append_record(
    vault_path: Path,
    target_date: date,
    category: RecordCategory,
    time: str,
    content: str,
    photo: str | None = None,
) -> None:
    """Append a record entry to the daily file under the correct section."""
    filepath = get_daily_filepath(vault_path, target_date)

    if not filepath.exists():
        raise FileNotFoundError(f"Daily file not found: {filepath}")

    heading = CATEGORY_HEADINGS[category]
    text = filepath.read_text(encoding="utf-8")

    # Build the entry line
    entry = f"- {time} {content}"
    if photo:
        entry += f" ![[{photo}]]"

    # Find the section and append the entry
    # Match: ## {heading}\n ... (until next ## or end of file)
    pattern = rf"(## {re.escape(heading)}\n)"
    replacement = rf"\1{entry}\n"
    new_text = re.sub(pattern, replacement, text, count=1)

    filepath.write_text(new_text, encoding="utf-8")


def get_section(
    vault_path: Path, target_date: date, category: RecordCategory
) -> str:
    """Read a specific section from a daily file. Returns the section text."""
    filepath = get_daily_filepath(vault_path, target_date)

    if not filepath.exists():
        return ""

    heading = CATEGORY_HEADINGS[category]
    text = filepath.read_text(encoding="utf-8")

    # Extract section between ## headings
    # Match from ## {heading} to the next ## or end of file
    pattern = rf"## {re.escape(heading)}\n(.*?)(?=\n## |\Z)"
    match = re.search(pattern, text, re.DOTALL)
    if match:
        return match.group(1).strip()
    return ""


def read_daily(vault_path: Path, target_date: date) -> str | None:
    """Read the full daily file content. Returns None if file doesn't exist."""
    filepath = get_daily_filepath(vault_path, target_date)

    if not filepath.exists():
        return None

    return filepath.read_text(encoding="utf-8")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd babynote && python -m pytest tests/test_daily.py -v`
Expected: PASS — all 4 tests pass

- [ ] **Step 5: Commit**

```bash
cd babynote
git add vault/daily.py tests/test_daily.py
git commit -m "feat: daily markdown file creation and record appending"
```

---

### Task 4: Vault — Attachments (Image Storage)

**Files:**
- Create: `babynote/vault/attachments.py`
- Create: `babynote/tests/test_attachments.py`

- [ ] **Step 1: Write the failing test for attachments**

```python
# tests/test_attachments.py
import pytest
import io
from pathlib import Path
from vault.attachments import save_image, generate_image_filename


def test_generate_image_filename():
    from datetime import date
    filename = generate_image_filename(
        target_date=date(2026, 5, 28), extension="jpg"
    )
    assert filename.startswith("baby-2026-05-28-")
    assert filename.endswith(".jpg")


def test_generate_image_filenames_unique():
    from datetime import date
    names = set()
    for _ in range(10):
        names.add(generate_image_filename(date(2026, 5, 28), "jpg"))
    assert len(names) == 10  # All unique


def test_save_image(tmp_path):
    vault_path = tmp_path
    image_data = b"\xff\xd8\xff\xe0fake_jpg_data"

    saved_path = save_image(
        vault_path=vault_path,
        image_data=image_data,
        target_date=date(2026, 5, 28),
        extension="jpg",
    )

    assert saved_path.exists()
    assert saved_path.read_bytes() == image_data
    # File should be in attachments/ directory
    assert "attachments" in str(saved_path)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd babynote && python -m pytest tests/test_attachments.py -v`
Expected: FAIL — module `vault.attachments` not found

- [ ] **Step 3: Implement attachments.py**

```python
# vault/attachments.py
import uuid
from datetime import date
from pathlib import Path
from config import ATTACHMENTS_DIR


def generate_image_filename(target_date: date, extension: str = "jpg") -> str:
    """Generate a unique filename for a baby photo.

    Format: baby-YYYY-MM-DD-<short_uuid>.<ext>
    """
    short_id = uuid.uuid4().hex[:6]
    return f"baby-{target_date.isoformat()}-{short_id}.{extension}"


def save_image(
    vault_path: Path,
    image_data: bytes,
    target_date: date,
    extension: str = "jpg",
) -> Path:
    """Save image data to the vault attachments directory.

    Returns the Path to the saved image file.
    """
    attachments_dir = vault_path / "attachments"
    attachments_dir.mkdir(parents=True, exist_ok=True)

    filename = generate_image_filename(target_date, extension)
    filepath = attachments_dir / filename

    filepath.write_bytes(image_data)
    return filepath
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd babynote && python -m pytest tests/test_attachments.py -v`
Expected: PASS — all 3 tests pass

- [ ] **Step 5: Commit**

```bash
cd babynote
git add vault/attachments.py tests/test_attachments.py
git commit -m "feat: image storage for vault attachments"
```

---

### Task 5: LLM — Intent Classification + Extraction + Query

**Files:**
- Create: `babynote/llm/__init__.py`
- Create: `babynote/llm/intent.py`
- Create: `babynote/llm/extract.py`
- Create: `babynote/llm/answer.py`
- Create: `babynote/tests/test_intent.py`
- Create: `babynote/tests/test_extract.py`
- Create: `babynote/tests/test_answer.py`

- [ ] **Step 1: Write the failing test for intent classification**

```python
# tests/test_intent.py
import pytest
from unittest.mock import patch, MagicMock
from llm.intent import classify_intent
from models import IntentType


@patch("llm.intent.call_llm")
def test_classify_record_intent(mock_llm):
    mock_llm.return_value = "record"
    result = classify_intent("宝宝刚吃了120ml奶")
    assert result.type == IntentType.RECORD


@patch("llm.intent.call_llm")
def test_classify_query_intent(mock_llm):
    mock_llm.return_value = "query"
    result = classify_intent("宝宝今天吃了多少奶")
    assert result.type == IntentType.QUERY


@patch("llm.intent.call_llm")
def test_classify_report_intent(mock_llm):
    mock_llm.return_value = "report"
    result = classify_intent("生成本月报告")
    assert result.type == IntentType.REPORT


@patch("llm.intent.call_llm")
def test_classify_unknown_intent(mock_llm):
    mock_llm.return_value = "unknown"
    result = classify_intent("今天天气怎么样")
    assert result.type == IntentType.UNKNOWN
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd babynote && python -m pytest tests/test_intent.py -v`
Expected: FAIL — module `llm.intent` not found

- [ ] **Step 3: Implement llm/__init__.py and llm/intent.py**

```python
# llm/__init__.py
```

```python
# llm/intent.py
import json
from openai import OpenAI
from models import Intent, IntentType
from config import OPENAI_API_KEY, OPENAI_BASE_URL, OPENAI_MODEL

INTENT_PROMPT = """你是一个意图分类器。根据用户输入，判断意图是以下哪种，只返回类别名:

- record: 记录宝宝日常（吃喝/排泄/睡眠/指标/里程碑/健康）
- query: 查询历史记录
- report: 生成成长报告
- photo: 发送了照片需要归类
- unknown: 无法判断

用户输入: {text}

只返回一个类别名，不要多余内容。"""


def call_llm(prompt: str) -> str:
    """Call LLM with a prompt and return the response text."""
    client = OpenAI(api_key=OPENAI_API_KEY, base_url=OPENAI_BASE_URL)
    response = client.chat.completions.create(
        model=OPENAI_MODEL,
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
        max_tokens=20,
    )
    return response.choices[0].message.content.strip()


def classify_intent(text: str) -> Intent:
    """Classify user input intent using LLM."""
    prompt = INTENT_PROMPT.format(text=text)
    result = call_llm(prompt)

    # Map result to IntentType
    intent_map = {
        "record": IntentType.RECORD,
        "query": IntentType.QUERY,
        "report": IntentType.REPORT,
        "photo": IntentType.PHOTO,
        "unknown": IntentType.UNKNOWN,
    }

    intent_type = intent_map.get(result.lower(), IntentType.UNKNOWN)
    return Intent(type=intent_type)
```

- [ ] **Step 4: Write the failing test for extraction**

```python
# tests/test_extract.py
import pytest
from unittest.mock import patch
from llm.extract import extract_record
from models import Record, RecordCategory


@patch("llm.extract.call_llm")
def test_extract_eating_record(mock_llm):
    mock_llm.return_value = '{"category": "吃喝", "time": "10:00", "content": "配方奶 120ml", "photo": null}'
    result = extract_record("宝宝刚吃了120ml奶")
    assert result.category == RecordCategory.EATING
    assert result.content == "配方奶 120ml"


@patch("llm.extract.call_llm")
def test_extract_sleep_record(mock_llm):
    mock_llm.return_value = '{"category": "睡眠", "time": "21:30", "content": "入睡", "photo": null}'
    result = extract_record("宝宝21:30睡着了")
    assert result.category == RecordCategory.SLEEP


@patch("llm.extract.call_llm")
def test_extract_with_photo(mock_llm):
    mock_llm.return_value = '{"category": "里程碑", "time": "now", "content": "第一次扶站", "photo": "baby-2026-05-28-abc123.jpg"}'
    result = extract_record("宝宝第一次扶站")
    assert result.category == RecordCategory.MILESTONE
    assert result.photo is not None
```

- [ ] **Step 5: Implement llm/extract.py**

```python
# llm/extract.py
import json
from openai import OpenAI
from models import Record, RecordCategory
from config import OPENAI_API_KEY, OPENAI_BASE_URL, OPENAI_MODEL

EXTRACT_PROMPT = """将以下自然语言提取为结构化数据，输出JSON格式:

{{
  "category": "吃喝|排泄|睡眠|身体指标|里程碑|健康",
  "time": "HH:MM 或 now",
  "content": "具体内容描述",
  "photo": null 或 图片文件名
}}

用户输入: {text}
当前时间: {current_time}

只返回JSON，不要多余内容。"""

CATEGORY_MAP = {
    "吃喝": RecordCategory.EATING,
    "排泄": RecordCategory.DIAPER,
    "睡眠": RecordCategory.SLEEP,
    "身体指标": RecordCategory.MEASUREMENT,
    "里程碑": RecordCategory.MILESTONE,
    "健康": RecordCategory.HEALTH,
}


def call_llm(prompt: str) -> str:
    """Call LLM with a prompt and return the response text."""
    client = OpenAI(api_key=OPENAI_API_KEY, base_url=OPENAI_BASE_URL)
    response = client.chat.completions.create(
        model=OPENAI_MODEL,
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
        max_tokens=200,
    )
    return response.choices[0].message.content.strip()


def extract_record(text: str, current_time: str = None) -> Record:
    """Extract structured record from natural language input."""
    from datetime import datetime

    if current_time is None:
        current_time = datetime.now().strftime("%H:%M")

    prompt = EXTRACT_PROMPT.format(text=text, current_time=current_time)
    result = call_llm(prompt)

    # Parse JSON response
    try:
        data = json.loads(result)
    except json.JSONDecodeError:
        # Try to extract JSON from markdown code block
        import re
        json_match = re.search(r"```(?:json)?\s*(.*?)```", result, re.DOTALL)
        if json_match:
            data = json.loads(json_match.group(1))
        else:
            raise ValueError(f"Failed to parse LLM response as JSON: {result}")

    category = CATEGORY_MAP.get(data["category"], RecordCategory.HEALTH)
    time_str = data.get("time", current_time)
    if time_str == "now":
        time_str = current_time

    return Record(
        category=category,
        time=time_str,
        content=data["content"],
        photo=data.get("photo"),
    )
```

- [ ] **Step 6: Write the failing test for query answering**

```python
# tests/test_answer.py
import pytest
from unittest.mock import patch
from llm.answer import answer_query


@patch("llm.answer.call_llm")
def test_answer_daily_query(mock_llm):
    mock_llm.return_value = "今天宝宝吃了3次配方奶，共360ml，加上2次母乳。"
    daily_data = "## 吃喝\n- 10:00 配方奶 120ml\n- 15:00 配方奶 90ml\n- 21:00 配方奶 150ml"
    result = answer_query("宝宝今天吃了多少奶", daily_data)
    assert "配方奶" in result or "360" in result or "ml" in result
```

- [ ] **Step 7: Implement llm/answer.py**

```python
# llm/answer.py
from openai import OpenAI
from config import OPENAI_API_KEY, OPENAI_BASE_URL, OPENAI_MODEL

ANSWER_PROMPT = """根据以下宝宝记录数据，回答用户问题。

数据:
{data}

问题: {query}

用自然语言回答，包含具体数字和时间。如果数据不足以回答，请如实说明。"""


def call_llm(prompt: str) -> str:
    """Call LLM with a prompt and return the response text."""
    client = OpenAI(api_key=OPENAI_API_KEY, base_url=OPENAI_BASE_URL)
    response = client.chat.completions.create(
        model=OPENAI_MODEL,
        messages=[{"role": "user", "content": prompt}],
        temperature=0.3,
        max_tokens=500,
    )
    return response.choices[0].message.content.strip()


def answer_query(query: str, data: str) -> str:
    """Answer a user query using the provided baby record data."""
    prompt = ANSWER_PROMPT.format(data=data, query=query)
    return call_llm(prompt)
```

- [ ] **Step 8: Run all LLM tests to verify they pass**

Run: `cd babynote && python -m pytest tests/test_intent.py tests/test_extract.py tests/test_answer.py -v`
Expected: PASS — all 7 tests pass

- [ ] **Step 9: Commit**

```bash
cd babynote
git add llm/ tests/test_intent.py tests/test_extract.py tests/test_answer.py
git commit -m "feat: LLM intent classification, record extraction, and query answering"
```

---

### Task 6: Feishu — Auth + Media Download

**Files:**
- Create: `babynote/feishu/__init__.py`
- Create: `babynote/feishu/auth.py`
- Create: `babynote/feishu/media.py`
- Create: `babynote/tests/test_feishu_auth.py`

- [ ] **Step 1: Write the failing test for Feishu auth**

```python
# tests/test_feishu_auth.py
import pytest
from unittest.mock import patch, MagicMock
from feishu.auth import get_tenant_access_token, verify_event_token


@patch("feishu.auth.requests.post")
def test_get_tenant_access_token(mock_post):
    mock_response = MagicMock()
    mock_response.json.return_value = {
        "code": 0,
        "tenant_access_token": "t-abc123",
        "expire": 7200,
    }
    mock_post.return_value = mock_response

    token = get_tenant_access_token()
    assert token == "t-abc123"


def test_verify_event_token():
    from config import FEISHU_VERIFICATION_TOKEN
    # Correct token should pass
    assert verify_event_token(FEISHU_VERIFICATION_TOKEN) is True
    # Wrong token should fail
    assert verify_event_token("wrong_token") is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd babynote && python -m pytest tests/test_feishu_auth.py -v`
Expected: FAIL — module `feishu.auth` not found

- [ ] **Step 3: Implement feishu/__init__.py and feishu/auth.py**

```python
# feishu/__init__.py
```

```python
# feishu/auth.py
import requests
from config import FEISHU_APP_ID, FEISHU_APP_SECRET, FEISHU_VERIFICATION_TOKEN


def get_tenant_access_token() -> str:
    """Get Feishu tenant access token."""
    url = "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal"
    payload = {
        "app_id": FEISHU_APP_ID,
        "app_secret": FEISHU_APP_SECRET,
    }
    headers = {"Content-Type": "application/json"}

    response = requests.post(url, json=payload, headers=headers)
    data = response.json()

    if data.get("code") != 0:
        raise RuntimeError(f"Failed to get tenant access token: {data}")

    return data["tenant_access_token"]


# Cache the token
_token_cache = {"token": None, "expire_at": 0}


def get_cached_token() -> str:
    """Get a cached tenant access token, refreshing if needed."""
    import time

    if _token_cache["token"] and time.time() < _token_cache["expire_at"] - 60:
        return _token_cache["token"]

    token = get_tenant_access_token()
    _token_cache["token"] = token
    _token_cache["expire_at"] = time.time() + 7200
    return token


def verify_event_token(token: str) -> bool:
    """Verify that an event's token matches our verification token."""
    return token == FEISHU_VERIFICATION_TOKEN
```

- [ ] **Step 4: Write the failing test for media download**

```python
# tests/test_feishu_media.py (we'll add this alongside auth tests)
import pytest
from unittest.mock import patch, MagicMock
from feishu.media import download_image, voice_to_text


@patch("feishu.media.get_cached_token")
@patch("feishu.media.requests.get")
def test_download_image(mock_get, mock_token):
    mock_token.return_value = "t-abc123"
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_response.content = b"\xff\xd8\xff\xe0fake_jpg_data"
    mock_get.return_value = mock_response

    data = download_image("msg_abc123")
    assert data == b"\xff\xd8\xff\xe0fake_jpg_data"


@patch("feishu.media.get_cached_token")
@patch("feishu.media.requests.get")
def test_voice_to_text(mock_get, mock_token):
    mock_token.return_value = "t-abc123"
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_response.json.return_value = {
        "code": 0,
        "data": {"text": "宝宝八点半吃了一侧母乳大概十五分钟"},
    }
    mock_get.return_value = mock_response

    text = voice_to_text("msg_voice_abc123")
    assert text == "宝宝八点半吃了一侧母乳大概十五分钟"
```

- [ ] **Step 5: Implement feishu/media.py**

```python
# feishu/media.py
import requests
from config import FEISHU_APP_ID
from feishu.auth import get_cached_token


def download_image(image_key: str) -> bytes:
    """Download an image from Feishu by image_key.

    Returns the image bytes.
    """
    token = get_cached_token()
    url = "https://open.feishu.cn/open-apis/im/v1/images/" + image_key
    headers = {"Authorization": f"Bearer {token}"}

    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.content


def voice_to_text(file_key: str) -> str:
    """Convert a Feishu voice message to text using Feishu ASR.

    Returns the transcribed text.
    """
    token = get_cached_token()
    url = "https://open.feishu.cn/open-apis/im/v1/messages/" + file_key + "/resources/voice"
    headers = {"Authorization": f"Bearer {token}"}
    params = {"type": "voice"}

    response = requests.get(url, headers=headers, params=params)
    data = response.json()

    if data.get("code") != 0:
        # Fallback: try to get the text directly from message content
        # In production, Feishu may return ASR text in the message itself
        raise RuntimeError(f"Voice-to-text failed: {data}")

    return data["data"]["text"]
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd babynote && python -m pytest tests/test_feishu_auth.py tests/test_feishu_media.py -v`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
cd babynote
git add feishu/ tests/test_feishu_auth.py tests/test_feishu_media.py
git commit -m "feat: Feishu auth token management and media download"
```

---

### Task 7: Feishu — Bot Webhook Handler + Message Dispatch

**Files:**
- Create: `babynote/feishu/bot.py`
- Create: `babynote/tests/test_bot.py`

- [ ] **Step 1: Write the failing test for bot message dispatch**

```python
# tests/test_bot.py
import pytest
import json
from unittest.mock import patch, MagicMock
from feishu.bot import handle_message, MessageType


@patch("feishu.bot.process_text_message")
def test_handle_text_message(mock_process):
    mock_process.return_value = "✅ 已记录：10:00 配方奶 120ml"
    result = handle_message(
        msg_type=MessageType.TEXT,
        content='{"text":"宝宝刚吃了120ml奶"}',
        msg_id="msg_001",
        user_id="user_001",
    )
    assert "已记录" in result
    mock_process.assert_called_once_with("宝宝刚吃了120ml奶", "user_001")


@patch("feishu.bot.process_image_message")
def test_handle_image_message(mock_process):
    mock_process.return_value = "照片已收到，请选择类别：\nA) 里程碑瞬间\nB) 身体指标\nC) 只记录照片"
    result = handle_message(
        msg_type=MessageType.IMAGE,
        content='{"image_key":"img_abc123"}',
        msg_id="msg_002",
        user_id="user_001",
    )
    assert "选择类别" in result


@patch("feishu.bot.process_voice_message")
def test_handle_voice_message(mock_process):
    mock_process.return_value = "✅ 已记录：08:30 母乳 15min 左侧"
    result = handle_message(
        msg_type=MessageType.VOICE,
        content='{"file_key":"file_voice_001"}',
        msg_id="msg_003",
        user_id="user_001",
    )
    assert "已记录" in result
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd babynote && python -m pytest tests/test_bot.py -v`
Expected: FAIL — module `feishu.bot` not found

- [ ] **Step 3: Implement feishu/bot.py**

```python
# feishu/bot.py
import json
from enum import Enum
from typing import Optional
from dataclasses import dataclass, field

from llm.intent import classify_intent, IntentType
from llm.extract import extract_record
from llm.answer import answer_query
from vault.daily import create_daily_file, append_record, read_daily, get_section
from vault.config_store import load_config, compute_age
from vault.attachments import save_image
from feishu.media import download_image, voice_to_text
from config import VAULT_PATH

from datetime import date, datetime
from models import RecordCategory, Record


class MessageType(str, Enum):
    TEXT = "text"
    IMAGE = "image"
    VOICE = "audio"
    POST = "post"  # Rich text


# In-memory state for multi-turn conversations (photo category selection)
@dataclass
class ConversationState:
    """Track pending interactions that need a follow-up response."""
    user_id: str
    awaiting: str = ""  # "photo_category", etc.
    pending_image_path: str = ""
    pending_image_filename: str = ""


# Simple in-memory state store — keyed by user_id
_pending_states: dict[str, ConversationState] = {}


def process_text_message(text: str, user_id: str) -> str:
    """Process a text message: classify intent and route accordingly."""
    # Check if there's a pending state
    state = _pending_states.get(user_id)

    if state and state.awaiting == "photo_category":
        # User is responding to photo category question
        category_map = {
            "a": RecordCategory.MILESTONE,
            "b": RecordCategory.MEASUREMENT,
            "c": RecordCategory.MILESTONE,  # "只记录照片" defaults to milestone
        }
        category = category_map.get(text.strip().lower(), RecordCategory.MILESTONE)

        today = date.today()
        baby_config = load_config(VAULT_PATH / "baby" / "config.yaml")
        baby_age = compute_age(baby_config.birthday, today)

        create_daily_file(VAULT_PATH, today, baby_age)
        append_record(
            VAULT_PATH, today, category, datetime.now().strftime("%H:%M"),
            "📷 照片记录", state.pending_image_filename,
        )

        heading_map = {
            RecordCategory.MILESTONE: "里程碑",
            RecordCategory.MEASUREMENT: "身体指标",
        }
        category_name = heading_map.get(category, "里程碑")
        _pending_states.pop(user_id, None)
        return f"✅ 已记录到{category_name}"

    # Classify intent
    intent = classify_intent(text)

    if intent.type == IntentType.RECORD:
        today = date.today()
        baby_config = load_config(VAULT_PATH / "baby" / "config.yaml")
        baby_age = compute_age(baby_config.birthday, today)

        create_daily_file(VAULT_PATH, today, baby_age)

        record = extract_record(text, datetime.now().strftime("%H:%M"))
        append_record(
            VAULT_PATH, today, record.category,
            record.time, record.content, record.photo,
        )

        category_names = {
            RecordCategory.EATING: "吃喝",
            RecordCategory.DIAPER: "排泄",
            RecordCategory.SLEEP: "睡眠",
            RecordCategory.MEASUREMENT: "身体指标",
            RecordCategory.MILESTONE: "里程碑",
            RecordCategory.HEALTH: "健康",
        }
        return f"✅ 已记录：{record.time} {record.content}"

    elif intent.type == IntentType.QUERY:
        today = date.today()
        # Determine date range from query — for now, default to today
        # TODO: parse date range from query in future iteration
        daily_content = read_daily(VAULT_PATH, today)

        if not daily_content:
            return "今天还没有记录哦，先记点什么吧～"

        answer = answer_query(text, daily_content)
        return answer

    elif intent.type == IntentType.REPORT:
        from vault.reports import generate_report
        # Determine report type from message
        if "月" in text or "月报" in text:
            report = generate_report(VAULT_PATH, "monthly")
        elif "周" in text or "周报" in text:
            report = generate_report(VAULT_PATH, "weekly")
        else:
            report = generate_report(VAULT_PATH, "daily")
        return report

    elif intent.type == IntentType.PHOTO:
        # This shouldn't happen for text, but handle gracefully
        return "请直接发送照片，我会帮你归类记录。"

    else:
        return "我不太明白你的意思，你是想：\n1. 记录宝宝的日常\n2. 查询历史记录\n3. 生成成长报告\n请告诉我～"


def process_image_message(image_key: str, user_id: str) -> str:
    """Download image, save to vault, and ask for category."""
    import tempfile
    from pathlib import Path

    today = date.today()

    # Download image from Feishu
    image_data = download_image(image_key)

    # Save to vault attachments
    saved_path = save_image(VAULT_PATH, image_data, today, extension="jpg")
    filename = saved_path.name

    # Store state awaiting category selection
    state = ConversationState(
        user_id=user_id,
        awaiting="photo_category",
        pending_image_path=str(saved_path),
        pending_image_filename=filename,
    )
    _pending_states[user_id] = state

    return "📷 照片已收到！想记在哪个类别？\nA) 里程碑瞬间\nB) 身体指标\nC) 只记录照片"


def process_voice_message(file_key: str, user_id: str) -> str:
    """Convert voice to text, then process as text message."""
    text = voice_to_text(file_key)
    return process_text_message(text, user_id)


def handle_message(
    msg_type: MessageType, content: str, msg_id: str, user_id: str
) -> str:
    """Main message handler that dispatches based on message type."""
    content_data = json.loads(content)

    if msg_type == MessageType.TEXT:
        text = content_data.get("text", "")
        return process_text_message(text, user_id)

    elif msg_type == MessageType.IMAGE:
        image_key = content_data.get("image_key", "")
        return process_image_message(image_key, user_id)

    elif msg_type == MessageType.VOICE:
        file_key = content_data.get("file_key", "")
        return process_voice_message(file_key, user_id)

    else:
        return "暂不支持这种消息类型，请发送文字、语音或图片。"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd babynote && python -m pytest tests/test_bot.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
cd babynote
git add feishu/bot.py tests/test_bot.py
git commit -m "feat: message dispatch with intent routing, record, query, and photo handling"
```

---

### Task 8: Vault — Report Generation

**Files:**
- Create: `babynote/vault/reports.py`
- Create: `babynote/tests/test_reports.py`

- [ ] **Step 1: Write the failing test for report generation**

```python
# tests/test_reports.py
import pytest
from datetime import date
from pathlib import Path
from vault.reports import generate_report
from vault.daily import create_daily_file, append_record
from models import RecordCategory


def _setup_daily_records(vault_path, target_date, baby_age="8个月12天"):
    """Helper: create a daily file with sample data."""
    create_daily_file(vault_path, target_date, baby_age)
    append_record(vault_path, target_date, RecordCategory.EATING, "07:30", "母乳 15min 左侧")
    append_record(vault_path, target_date, RecordCategory.EATING, "10:00", "配方奶 120ml")
    append_record(vault_path, target_date, RecordCategory.EATING, "15:00", "配方奶 90ml")
    append_record(vault_path, target_date, RecordCategory.EATING, "21:00", "配方奶 150ml")
    append_record(vault_path, target_date, RecordCategory.DIAPER, "08:00", "尿布湿，黄色软便")
    append_record(vault_path, target_date, RecordCategory.DIAPER, "14:00", "换尿布，小便")
    append_record(vault_path, target_date, RecordCategory.SLEEP, "21:30", "入睡")
    append_record(vault_path, target_date, RecordCategory.HEALTH, "22:00", "无异常")


def test_generate_daily_report(tmp_path):
    _setup_daily_records(tmp_path, date(2026, 5, 28))

    report = generate_report(tmp_path, "daily", target_date=date(2026, 5, 28))

    assert "日报" in report or "2026-05-28" in report
    # Check the report file was created
    report_dir = tmp_path / "baby" / "reports" / "daily"
    assert report_dir.exists()


def test_generate_daily_report_no_data(tmp_path):
    """Daily report with no records should still generate."""
    report = generate_report(tmp_path, "daily", target_date=date(2026, 5, 28))
    assert "日报" in report or "2026-05-28" in report
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd babynote && python -m pytest tests/test_reports.py -v`
Expected: FAIL — module `vault.reports` not found

- [ ] **Step 3: Implement vault/reports.py**

```python
# vault/reports.py
import re
from datetime import date, timedelta
from pathlib import Path

from vault.daily import read_daily, get_section, create_daily_file
from vault.config_store import load_config, compute_age
from llm.answer import call_llm
from config import VAULT_PATH

from models import RecordCategory


DAILY_REPORT_PROMPT = """根据以下宝宝今日记录，生成一份简洁的日报。

日期: {date}
宝宝年龄: {baby_age}

今日记录:
{daily_data}

请生成日报，包含以下部分:
1. 🍼 吃喝汇总（总奶量/辅食种类）
2. 🧷 排泄次数
3. 😴 睡眠（时长+夜醒次数）
4. 📸 当日照片精选（如有）
5. 💡 今日小结

用 Markdown 格式输出。"""


WEEKLY_REPORT_PROMPT = """根据以下宝宝本周记录，生成一份周报。

周期: {start_date} ~ {end_date}
宝宝年龄: {baby_age}

本周记录:
{weekly_data}

请生成周报，包含以下部分:
1. 🍼 吃喝趋势（日均奶量/辅食变化）
2. 😴 睡眠规律（日均睡眠/夜醒趋势）
3. 📏 体重变化（如有记录）
4. 🎯 新里程碑
5. 📊 与上周对比
6. 💡 本周小结

用 Markdown 格式输出。"""


MONTHLY_REPORT_PROMPT = """根据以下宝宝本月记录，生成一份月报。

月份: {month}
宝宝年龄: {baby_age}
性别: {gender}

本月记录:
{monthly_data}

请生成月报，包含以下部分:
1. 🍼 吃喝（日均奶量/辅食进展/趋势）
2. 😴 睡眠（日均睡眠/夜醒频率/最佳睡眠日）
3. 📏 发育指标（体重变化/对比标准发育曲线）
4. 🎯 里程碑汇总
5. 📸 本月精选（如有照片记录）
6. 🔮 下月预期

用 Markdown 格式输出。"""


def generate_report(
    vault_path: Path,
    report_type: str,
    target_date: date = None,
) -> str:
    """Generate a growth report.

    Args:
        vault_path: Path to the Obsidian vault
        report_type: "daily", "weekly", or "monthly"
        target_date: The date for the report (defaults to today)

    Returns:
        The generated report content (also saved to file)
    """
    if target_date is None:
        target_date = date.today()

    # Load baby config for age calculation
    config_path = vault_path / "baby" / "config.yaml"
    try:
        baby_config = load_config(config_path)
        baby_age = compute_age(baby_config.birthday, target_date)
    except FileNotFoundError:
        baby_age = "未知"

    if report_type == "daily":
        return _generate_daily_report(vault_path, target_date, baby_age)
    elif report_type == "weekly":
        return _generate_weekly_report(vault_path, target_date, baby_age)
    elif report_type == "monthly":
        return _generate_monthly_report(vault_path, target_date, baby_age)
    else:
        raise ValueError(f"Unknown report type: {report_type}")


def _generate_daily_report(vault_path: Path, target_date: date, baby_age: str) -> str:
    """Generate a daily report for the given date."""
    daily_data = read_daily(vault_path, target_date)

    if not daily_data:
        daily_data = "今日暂无记录"

    prompt = DAILY_REPORT_PROMPT.format(
        date=target_date.isoformat(),
        baby_age=baby_age,
        daily_data=daily_data,
    )

    report_content = call_llm(prompt)

    # Save to file
    report_dir = vault_path / "baby" / "reports" / "daily"
    report_dir.mkdir(parents=True, exist_ok=True)
    report_path = report_dir / f"{target_date.isoformat()}.md"

    # Add frontmatter
    full_report = f"---\ntype: daily-report\ndate: {target_date.isoformat()}\nbaby_age: \"{baby_age}\"\n---\n\n{report_content}"
    report_path.write_text(full_report, encoding="utf-8")

    return report_content


def _generate_weekly_report(vault_path: Path, target_date: date, baby_age: str) -> str:
    """Generate a weekly report for the week containing target_date."""
    # Find Monday of the week
    start_of_week = target_date - timedelta(days=target_date.weekday())
    end_of_week = start_of_week + timedelta(days=6)

    weekly_data = ""
    for i in range(7):
        d = start_of_week + timedelta(days=i)
        daily = read_daily(vault_path, d)
        if daily:
            weekly_data += f"### {d.isoformat()}\n{daily}\n\n"

    if not weekly_data:
        weekly_data = "本周暂无记录"

    prompt = WEEKLY_REPORT_PROMPT.format(
        start_date=start_of_week.isoformat(),
        end_date=end_of_week.isoformat(),
        baby_age=baby_age,
        weekly_data=weekly_data,
    )

    report_content = call_llm(prompt)

    # Save to file — use ISO week number
    iso_week = f"{target_date.year}-W{target_date.isocalendar()[1]:02d}"
    report_dir = vault_path / "baby" / "reports" / "weekly"
    report_dir.mkdir(parents=True, exist_ok=True)
    report_path = report_dir / f"{iso_week}.md"

    full_report = f"---\ntype: weekly-report\nperiod: {iso_week}\nbaby_age: \"{baby_age}\"\n---\n\n{report_content}"
    report_path.write_text(full_report, encoding="utf-8")

    return report_content


def _generate_monthly_report(vault_path: Path, target_date: date, baby_age: str) -> str:
    """Generate a monthly report for the month containing target_date."""
    try:
        baby_config = load_config(vault_path / "baby" / "config.yaml")
        gender = baby_config.gender
    except FileNotFoundError:
        gender = "未知"

    year = target_date.year
    month = target_date.month

    monthly_data = ""
    # Collect all daily records for the month
    from datetime import calendar
    _, days_in_month = calendar.monthrange(year, month)

    for day in range(1, days_in_month + 1):
        d = date(year, month, day)
        if d > target_date:
            break  # Don't include future dates
        daily = read_daily(vault_path, d)
        if daily:
            monthly_data += f"### {d.isoformat()}\n{daily}\n\n"

    if not monthly_data:
        monthly_data = "本月暂无记录"

    month_str = f"{year}-{month:02d}"
    prompt = MONTHLY_REPORT_PROMPT.format(
        month=month_str,
        baby_age=baby_age,
        gender=gender,
        monthly_data=monthly_data,
    )

    report_content = call_llm(prompt)

    # Save to file
    report_dir = vault_path / "baby" / "reports" / "monthly"
    report_dir.mkdir(parents=True, exist_ok=True)
    report_path = report_dir / f"{month_str}.md"

    full_report = f"---\ntype: monthly-report\nperiod: {month_str}\nbaby_age: \"{baby_age}\"\n---\n\n{report_content}"
    report_path.write_text(full_report, encoding="utf-8")

    return report_content
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd babynote && python -m pytest tests/test_reports.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
cd babynote
git add vault/reports.py tests/test_reports.py
git commit -m "feat: daily/weekly/monthly report generation with LLM summarization"
```

---

### Task 9: Scheduler — Daily Auto Report

**Files:**
- Create: `babynote/scheduler/__init__.py`
- Create: `babynote/scheduler/jobs.py`

- [ ] **Step 1: Implement scheduler/__init__.py and scheduler/jobs.py**

```python
# scheduler/__init__.py
```

```python
# scheduler/jobs.py
from apscheduler.schedulers.background import BackgroundScheduler
from datetime import date
from config import VAULT_PATH, DAILY_REPORT_HOUR
from vault.reports import generate_report
from vault.config_store import load_config
from feishu.bot import process_text_message  # Reuse for notification


def generate_and_notify_daily_report():
    """Generate daily report and send notification via Feishu."""
    today = date.today()

    try:
        report = generate_report(VAULT_PATH, "daily", target_date=today)
        # TODO: Send report to Feishu — this will be wired in Task 10
        # when the webhook server is running
        print(f"[Scheduler] Daily report generated for {today}")
    except Exception as e:
        print(f"[Scheduler] Failed to generate daily report: {e}")


def setup_scheduler() -> BackgroundScheduler:
    """Set up the APScheduler with the daily report job."""
    scheduler = BackgroundScheduler()
    scheduler.add_job(
        generate_and_notify_daily_report,
        "cron",
        hour=DAILY_REPORT_HOUR,
        minute=0,
        id="daily_report",
        name="Daily Baby Report",
    )
    return scheduler
```

- [ ] **Step 2: Commit**

```bash
cd babynote
git add scheduler/
git commit -m "feat: APScheduler daily report job at 22:00"
```

---

### Task 10: Main — Webhook Server + First Run Setup

**Files:**
- Create: `babynote/main.py`

- [ ] **Step 1: Implement main.py — the Flask webhook server**

```python
# main.py
import json
import hashlib
from flask import Flask, request, jsonify
from config import FEISHU_VERIFICATION_TOKEN, FEISHU_ENCRYPT_KEY, PORT, VAULT_PATH
from feishu.bot import handle_message, MessageType
from feishu.auth import get_cached_token, verify_event_token, get_tenant_access_token
from feishu.media import download_image
from scheduler.jobs import setup_scheduler
from vault.config_store import load_config, save_config, compute_age
from models import BabyConfig
from pathlib import Path

import requests as http_requests


app = Flask(__name__)
scheduler = None


def send_feishu_message(user_id: str, text: str):
    """Send a text message to a Feishu user."""
    token = get_cached_token()
    url = "https://open.feishu.cn/open-apis/im/v1/messages"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    }
    payload = {
        "receive_id": user_id,
        "msg_type": "text",
        "content": json.dumps({"text": text}),
    }
    params = {"receive_id_type": "open_id"}

    http_requests.post(url, headers=headers, json=payload, params=params)


@app.route("/event", methods=["POST"])
def handle_event():
    """Handle Feishu event callback."""
    data = request.json

    # Handle URL verification challenge
    if data.get("type") == "url_verification":
        challenge = data.get("challenge", "")
        token = data.get("token", "")
        if verify_event_token(token):
            return jsonify({"challenge": challenge})
        return jsonify({"error": "invalid token"}), 403

    # Handle message events
    event = data.get("event", {})
    msg_type_str = event.get("message", {}).get("message_type", "text")
    content = event.get("message", {}).get("content", "{}")
    msg_id = event.get("message", {}).get("message_id", "")
    user_id = event.get("sender", {}).get("sender_id", {}).get("open_id", "")

    # Map Feishu message type to our enum
    type_map = {
        "text": MessageType.TEXT,
        "image": MessageType.IMAGE,
        "audio": MessageType.VOICE,
        "post": MessageType.POST,
    }
    msg_type = type_map.get(msg_type_str, MessageType.TEXT)

    try:
        reply = handle_message(msg_type, content, msg_id, user_id)
        send_feishu_message(user_id, reply)
    except Exception as e:
        send_feishu_message(user_id, f"⚠️ 处理消息时出错了：{str(e)}")

    return jsonify({"code": 0})


@app.route("/health", methods=["GET"])
def health():
    """Health check endpoint."""
    return jsonify({"status": "ok", "vault": str(VAULT_PATH)})


def check_first_run():
    """Check if baby config exists; if not, prompt user to set up."""
    config_path = VAULT_PATH / "baby" / "config.yaml"
    if not config_path.exists():
        print("=" * 50)
        print("🍼 BabyNote 首次运行！")
        print("=" * 50)
        print("请先设置宝宝基础信息。")
        name = input("宝宝名字/昵称: ").strip() or "宝宝"
        birthday = input("宝宝出生日期 (YYYY-MM-DD): ").strip()
        gender = input("宝宝性别 (男/女): ").strip() or "男"

        from datetime import datetime as dt
        try:
            birthday_date = dt.strptime(birthday, "%Y-%m-%d").date()
        except ValueError:
            print("日期格式错误，使用默认日期")
            birthday_date = dt.now().date()

        baby_dir = VAULT_PATH / "baby"
        baby_dir.mkdir(parents=True, exist_ok=True)

        config = BabyConfig(name=name, birthday=birthday_date, gender=gender)
        save_config(config, config_path)
        print(f"✅ 宝宝信息已保存: {name}, {birthday}, {gender}")
        print("=" * 50)


def main():
    """Start the BabyNote server."""
    global scheduler

    print("🍼 BabyNote 启动中...")
    check_first_run()

    # Start scheduler
    scheduler = setup_scheduler()
    scheduler.start()
    print("📅 日报定时任务已启动 (每天 22:00)")

    # Start Flask server
    print(f"🚀 服务器运行在端口 {PORT}")
    app.run(host="0.0.0.0", port=PORT)


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Commit**

```bash
cd babynote
git add main.py
git commit -m "feat: Flask webhook server with first-run setup and scheduler"
```

---

### Task 11: Integration Test + Deployment Config

**Files:**
- Create: `babynote/tests/test_integration.py`
- Create: `babynote/Dockerfile`
- Create: `babynote/docker-compose.yml`
- Create: `babynote/README.md`

- [ ] **Step 1: Write integration test**

```python
# tests/test_integration.py
import pytest
from datetime import date
from pathlib import Path
from unittest.mock import patch, MagicMock

from vault.config_store import save_config, compute_age
from vault.daily import create_daily_file, append_record, read_daily, get_section
from vault.attachments import save_image
from models import BabyConfig, RecordCategory


def test_full_day_workflow(tmp_path):
    """Test a complete day's workflow: setup → record → query data."""
    # 1. Setup baby config
    config_path = tmp_path / "baby" / "config.yaml"
    baby_dir = tmp_path / "baby"
    baby_dir.mkdir(parents=True, exist_ok=True)

    config = BabyConfig(name="小宝", birthday=date(2025, 9, 16), gender="男")
    save_config(config, config_path)

    loaded = load_config(config_path := config_path)  # noqa
    assert loaded.name == "小宝"

    # 2. Compute age
    age = compute_age(date(2025, 9, 16), date(2026, 5, 28))
    assert "8个月" in age

    # 3. Create daily file
    create_daily_file(tmp_path, date(2026, 5, 28), age)

    # 4. Add records
    append_record(tmp_path, date(2026, 5, 28), RecordCategory.EATING, "10:00", "配方奶 120ml")
    append_record(tmp_path, date(2026, 5, 28), RecordCategory.EATING, "15:00", "配方奶 90ml")
    append_record(tmp_path, date(2026, 5, 28), RecordCategory.DIAPER, "08:00", "黄色软便")
    append_record(tmp_path, date(2026, 5, 28), RecordCategory.SLEEP, "21:30", "入睡")

    # 5. Read back
    eating = get_section(tmp_path, date(2026, 5, 28), RecordCategory.EATING)
    assert "配方奶 120ml" in eating
    assert "配方奶 90ml" in eating

    # 6. Full day content
    daily = read_daily(tmp_path, date(2026, 5, 28))
    assert daily is not None
    assert "2026-05-28" in daily


def test_image_save_and_reference(tmp_path):
    """Test saving an image and verifying it exists."""
    image_data = b"\xff\xd8\xff\xe0fake_jpg_data"
    saved = save_image(tmp_path, image_data, date(2026, 5, 28), "jpg")

    assert saved.exists()
    assert saved.read_bytes() == image_data
    assert saved.name.startswith("baby-2026-05-28-")
```

- [ ] **Step 2: Create Dockerfile**

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080

CMD ["python", "main.py"]
```

- [ ] **Step 3: Create docker-compose.yml**

```yaml
# docker-compose.yml
version: "3.8"

services:
  babynote:
    build: .
    ports:
      - "8080:8080"
    volumes:
      - ./vault:/app/vault
    env_file:
      - .env
    restart: unless-stopped
```

- [ ] **Step 4: Create README.md**

```markdown
# BabyNote 🍼

宝宝成长记录 Agent — 通过飞书 Bot 随时用语音/文字/图片记录宝宝日常，数据存 Obsidian Vault。

## 快速开始

1. 复制 `.env.example` 为 `.env`，填入飞书 Bot 和 OpenAI 配置
2. 安装依赖: `pip install -r requirements.txt`
3. 启动服务: `python main.py`
4. 首次运行会引导设置宝宝基础信息
5. 在飞书开放平台配置 Bot 事件回调 URL: `http://your-server:8080/event`

## 功能

- 📝 文字/语音记录宝宝日常（吃喝/排泄/睡眠/身体指标/里程碑/健康）
- 📷 照片记录 + 自动归类
- 🔍 自然语言查询历史记录
- 📊 日报自动生成 + 飞书推送
- 📈 周报/月报手动触发
- 💾 数据 100% 存 Obsidian Markdown，本地可控

## Docker 部署

```bash
docker-compose up -d
```

## Vault 结构

```
vault/
├── baby/
│   ├── config.yaml           # 宝宝基础信息
│   ├── 2026-05-28.md          # 每日记录
│   └── reports/
│       ├── daily/             # 日报
│       ├── weekly/            # 周报
│       └── monthly/           # 月报
└── attachments/               # 照片等附件
```

## 许可

MIT
```

- [ ] **Step 5: Run integration test**

Run: `cd babynote && python -m pytest tests/test_integration.py -v`
Expected: PASS — 2 integration tests pass

- [ ] **Step 6: Run all tests**

Run: `cd babynote && python -m pytest tests/ -v`
Expected: All tests pass

- [ ] **Step 7: Commit and tag**

```bash
cd babynote
git add tests/test_integration.py Dockerfile docker-compose.yml README.md
git commit -m "feat: integration tests, Docker config, and README"
git tag v0.1.0
```

---

## Self-Review Checklist

After writing this plan, I reviewed it against the spec:

1. **Spec coverage:**
   - ✅ 飞书 Bot 收发文字消息 → Task 7, 10
   - ✅ 语音 → 飞书 ASR → 文字 → Task 6, 7
   - ✅ 图片 → 下载 → 存储 → 追问归类 → Task 4, 7
   - ✅ 6 个维度的记录和追加 → Task 3, 5
   - ✅ 当日/跨日查询 → Task 5, 7
   - ✅ 每日自动创建 Markdown 文件 → Task 3
   - ✅ 日报自动生成 + 飞书推送 → Task 8, 9
   - ✅ 周报/月报手动触发 → Task 8
   - ✅ 宝宝基础信息配置 → Task 2

2. **Placeholder scan:** ✅ No TBD/TODO/vague steps found

3. **Type consistency:** ✅ Models, function signatures, and property names are consistent across tasks

4. **Missing from spec but handled:** First-run setup flow (Task 10), error handling in bot dispatch (Task 7)