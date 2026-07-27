> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](testing.md)

# 測試指南

本文件提供在 Open Notebook 中撰寫測試的準則。測試對於維持程式碼品質與防止回歸問題至關重要。

## 測試理念

### 該測試什麼

聚焦於測試最重要的東西:

- **商業邏輯** - 核心領域模型及其操作
- **API 合約** - HTTP 端點行為與錯誤處理
- **關鍵工作流程** - 使用者依賴的端到端流程
- **資料持久性** - 資料庫操作與資料完整性
- **錯誤情境** - 系統如何優雅地處理失敗

### 不該測試什麼

不要浪費時間測試框架程式碼:

- 框架功能(FastAPI、React 等)
- 第三方函式庫的實作
- 沒有邏輯的簡單 getter/setter
- 檢視/呈現層渲染(除非包含邏輯)

## 測試結構

所有 Python 測試我們都使用具備 async 支援的 **pytest**:

```python
import pytest
from httpx import AsyncClient
from open_notebook.domain.notebook import Notebook

@pytest.mark.asyncio
async def test_create_notebook():
    """Test notebook creation."""
    notebook = Notebook(name="Test Notebook", description="Test description")
    await notebook.save()

    assert notebook.id is not None
    assert notebook.name == "Test Notebook"
    assert notebook.created is not None

@pytest.mark.asyncio
async def test_api_create_notebook():
    """Test notebook creation via API."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/api/notebooks",
            json={"name": "Test Notebook", "description": "Test description"}
        )
        assert response.status_code == 200
        data = response.json()
        assert data["name"] == "Test Notebook"
```

## 測試分類

### 1. 單元測試

單獨測試個別函式與方法:

```python
@pytest.mark.asyncio
async def test_notebook_validation():
    """Test that notebook name validation works."""
    with pytest.raises(InvalidInputError):
        Notebook(name="", description="test")

@pytest.mark.asyncio
async def test_notebook_archive():
    """Test notebook archiving."""
    notebook = Notebook(name="Test", description="")
    notebook.archive()
    assert notebook.archived is True
```

**位置**:`tests/unit/`

### 2. 整合測試

測試元件之間的互動與資料庫操作:

```python
@pytest.mark.asyncio
async def test_create_notebook_with_sources():
    """Test creating a notebook and adding sources."""
    notebook = await create_notebook(name="Research", description="")
    source = await add_source(notebook_id=notebook.id, url="https://example.com")

    retrieved = await get_notebook_with_sources(notebook.id)
    assert len(retrieved.sources) == 1
    assert retrieved.sources[0].id == source.id
```

**位置**:`tests/integration/`

### 3. API 測試

測試 HTTP 端點與錯誤回應:

```python
@pytest.mark.asyncio
async def test_get_notebooks_endpoint():
    """Test GET /notebooks endpoint."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/api/notebooks")
        assert response.status_code == 200
        data = response.json()
        assert isinstance(data, list)

@pytest.mark.asyncio
async def test_create_notebook_validation():
    """Test that invalid input is rejected."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/api/notebooks",
            json={"name": "", "description": ""}
        )
        assert response.status_code == 400
```

**位置**:`tests/api/`

### 4. 資料庫測試

測試資料持久性與查詢正確性:

```python
@pytest.mark.asyncio
async def test_save_and_retrieve_notebook():
    """Test saving and retrieving a notebook from database."""
    notebook = Notebook(name="Test", description="desc")
    await notebook.save()

    retrieved = await Notebook.get(notebook.id)
    assert retrieved.name == "Test"
    assert retrieved.description == "desc"

@pytest.mark.asyncio
async def test_query_by_criteria():
    """Test querying notebooks by criteria."""
    await create_notebook("Active", "")
    await create_notebook("Archived", "")

    active = await repo_query(
        "SELECT * FROM notebook WHERE archived = false"
    )
    assert len(active) >= 1
```

**位置**:`tests/database/`

## 執行測試

### 執行所有測試

```bash
uv run pytest
```

### 執行特定測試檔案

```bash
uv run pytest tests/test_notebooks.py
```

### 執行特定測試函式

```bash
uv run pytest tests/test_notebooks.py::test_create_notebook
```

### 執行並產生涵蓋率報告

```bash
uv run pytest --cov=open_notebook
```

### 只執行單元測試

```bash
uv run pytest tests/unit/
```

### 只執行整合測試

```bash
uv run pytest tests/integration/
```

### 以詳細模式執行測試

```bash
uv run pytest -v
```

### 執行測試並顯示輸出

```bash
uv run pytest -s
```

## 測試 Fixtures

使用 pytest fixtures 處理常見的設定與清理:

```python
import pytest

@pytest.fixture
async def test_notebook():
    """Create a test notebook."""
    notebook = Notebook(name="Test Notebook", description="Test description")
    await notebook.save()
    yield notebook
    await notebook.delete()

@pytest.fixture
async def api_client():
    """Create an API test client."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.fixture
async def test_notebook_with_sources(test_notebook):
    """Create a test notebook with sample sources."""
    source1 = Source(notebook_id=test_notebook.id, url="https://example.com")
    source2 = Source(notebook_id=test_notebook.id, url="https://example.org")
    await source1.save()
    await source2.save()

    test_notebook.sources = [source1, source2]
    yield test_notebook

    # Cleanup
    await source1.delete()
    await source2.delete()
```

## 最佳實踐

### 1. 撰寫具描述性的測試名稱

```python
# Good - clearly describes what is being tested
async def test_create_notebook_with_valid_name_succeeds():
    ...

# Bad - vague about what's being tested
async def test_notebook():
    ...
```

### 2. 使用 Docstring

```python
@pytest.mark.asyncio
async def test_vector_search_returns_sorted_results():
    """Test that vector search results are sorted by relevance score."""
    # Implementation
```

### 3. 測試邊界情況

```python
@pytest.mark.asyncio
async def test_search_with_empty_query():
    """Test that empty query raises error."""
    with pytest.raises(InvalidInputError):
        await vector_search("")

@pytest.mark.asyncio
async def test_search_with_very_long_query():
    """Test that very long query is handled."""
    long_query = "x" * 10000
    results = await vector_search(long_query)
    assert isinstance(results, list)

@pytest.mark.asyncio
async def test_search_with_special_characters():
    """Test that special characters are handled."""
    results = await vector_search("@#$%^&*()")
    assert isinstance(results, list)
```

### 4. 有效使用斷言

```python
# Good - specific assertions
assert notebook.name == "Test"
assert len(notebook.sources) == 3
assert notebook.created is not None

# Less good - too broad
assert notebook is not None
assert notebook  # ambiguous what's being tested
```

### 5. 同時測試成功與失敗情境

```python
@pytest.mark.asyncio
async def test_create_notebook_success():
    """Test successful notebook creation."""
    notebook = await create_notebook(name="Research", description="AI")
    assert notebook.id is not None
    assert notebook.name == "Research"

@pytest.mark.asyncio
async def test_create_notebook_empty_name_fails():
    """Test that empty name raises error."""
    with pytest.raises(InvalidInputError):
        await create_notebook(name="", description="")

@pytest.mark.asyncio
async def test_create_notebook_duplicate_fails():
    """Test that duplicate names are handled."""
    await create_notebook(name="Research", description="")
    with pytest.raises(DuplicateError):
        await create_notebook(name="Research", description="")
```

### 6. 保持測試獨立

```python
# Good - test is self-contained
@pytest.mark.asyncio
async def test_archive_notebook():
    notebook = Notebook(name="Test", description="")
    await notebook.save()
    await notebook.archive()
    assert notebook.archived is True

# Bad - depends on another test's state
@pytest.mark.asyncio
async def test_archive_existing_notebook():
    # Assumes test_create_notebook ran first
    await notebook.archive()  # notebook undefined
```

### 7. 使用 Fixtures 處理可重用的設定

```python
# Instead of repeating setup:
@pytest.fixture
async def client_with_auth(api_client, mock_auth):
    """Client with authentication set up."""
    api_client.headers.update({"Authorization": f"Bearer {mock_auth.token}"})
    yield api_client

@pytest.mark.asyncio
async def test_protected_endpoint(client_with_auth):
    """Test protected endpoint."""
    response = await client_with_auth.get("/api/protected")
    assert response.status_code == 200
```

## 涵蓋率目標

- 整體涵蓋率以 70% 以上為目標
- 關鍵商業邏輯的涵蓋率以 90% 以上為目標
- 不要執著於 100%——聚焦於有意義的測試
- 使用 `--cov` 旗標檢查涵蓋率:`uv run pytest --cov=open_notebook`

## 非同步測試模式

### 測試非同步函式

```python
@pytest.mark.asyncio
async def test_async_operation():
    """Test async function."""
    result = await some_async_function()
    assert result is not None
```

### 測試並行操作

```python
@pytest.mark.asyncio
async def test_concurrent_notebook_creation():
    """Test creating multiple notebooks concurrently."""
    tasks = [
        create_notebook(f"Notebook {i}", "")
        for i in range(10)
    ]
    notebooks = await asyncio.gather(*tasks)
    assert len(notebooks) == 10
    assert all(n.id for n in notebooks)
```

## 常見測試錯誤

### 錯誤:「event loop is closed」

解法:正確使用 async fixture:
```python
@pytest.fixture
async def notebook():  # Use async fixture
    notebook = Notebook(name="Test", description="")
    await notebook.save()
    yield notebook
    await notebook.delete()
```

### 錯誤:「object is not awaitable」

解法:確認你有使用 await:
```python
# Wrong
result = create_notebook("Test", "")

# Right
result = await create_notebook("Test", "")
```

---

**另請參閱:**
- [程式碼標準](code-standards.zh.md) - 程式碼格式與風格
- [貢獻指南](contributing.zh.md) - 整體貢獻工作流程
