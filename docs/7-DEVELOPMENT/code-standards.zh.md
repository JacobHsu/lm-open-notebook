> 🇹🇼 中文版(本頁)｜ 🇬🇧 [English](code-standards.md)

# 程式碼標準

本文件說明 Open Notebook 貢獻專案的程式碼標準與最佳實踐。所有程式碼都應遵循這些準則,以確保一致性、可讀性與可維護性。

## Python 標準

### 程式碼格式

我們遵循 **PEP 8**,並搭配一些特定準則:

- 使用 **Ruff** 進行 lint 與格式化
- 最大行長:**88 字元**
- 字串使用**雙引號**
- 多行結構使用**尾隨逗號**

### 型別提示

函式參數與回傳值務必使用型別提示:

```python
from typing import List, Optional, Dict, Any
from pydantic import BaseModel

async def process_content(
    content: str,
    options: Optional[Dict[str, Any]] = None
) -> ProcessedContent:
    """Process content with optional configuration."""
    # Implementation
```

### Async/Await 模式

在整個程式碼庫中一致使用 async/await:

```python
# Good
async def fetch_data(url: str) -> Dict[str, Any]:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

# Bad - mixing sync and async
def fetch_data(url: str) -> Dict[str, Any]:
    loop = asyncio.get_event_loop()
    return loop.run_until_complete(async_fetch(url))
```

### 錯誤處理

使用自訂例外進行結構化錯誤處理:

```python
from open_notebook.exceptions import DatabaseOperationError, InvalidInputError

async def create_notebook(name: str, description: str) -> Notebook:
    """Create a new notebook with validation."""
    if not name.strip():
        raise InvalidInputError("Notebook name cannot be empty")

    try:
        notebook = Notebook(name=name, description=description)
        await notebook.save()
        return notebook
    except Exception as e:
        raise DatabaseOperationError(f"Failed to create notebook: {str(e)}")
```

### 文件(Google 風格 Docstring)

所有函式、類別與模組都使用 Google 風格的 docstring:

```python
async def vector_search(
    query: str,
    limit: int = 10,
    minimum_score: float = 0.2
) -> List[SearchResult]:
    """Perform vector search across embedded content.

    Args:
        query: Search query string
        limit: Maximum number of results to return
        minimum_score: Minimum similarity score for results

    Returns:
        List of search results sorted by relevance score

    Raises:
        InvalidInputError: If query is empty or limit is invalid
        DatabaseOperationError: If search operation fails
    """
    # Implementation
```

#### 模組 Docstring
```python
"""
Notebook domain model and operations.

This module contains the core Notebook class and related operations for
managing research notebooks within the Open Notebook system.
"""
```

#### 類別 Docstring
```python
class Notebook(BaseModel):
    """A research notebook containing sources, notes, and chat sessions.

    Notebooks are the primary organizational unit in Open Notebook, allowing
    users to group related research materials and maintain separate contexts
    for different projects.

    Attributes:
        name: The notebook's display name
        description: Optional description of the notebook's purpose
        archived: Whether the notebook is archived (default: False)
        created: Timestamp of creation
        updated: Timestamp of last update
    """
```

#### 函式 Docstring
```python
async def create_notebook(
    name: str,
    description: str = "",
    user_id: Optional[str] = None
) -> Notebook:
    """Create a new notebook with validation.

    Args:
        name: The notebook name (required, non-empty)
        description: Optional notebook description
        user_id: Optional user ID for multi-user deployments

    Returns:
        The created notebook instance

    Raises:
        InvalidInputError: If name is empty or invalid
        DatabaseOperationError: If creation fails

    Example:
        ```python
        notebook = await create_notebook(
            name="AI Research",
            description="Research on AI applications"
        )
        ```
    """
```

## FastAPI 標準

### Router 組織方式

依領域組織端點:

```python
# api/routers/notebooks.py
from fastapi import APIRouter, HTTPException, Query
from typing import List, Optional

router = APIRouter()

@router.get("/notebooks", response_model=List[NotebookResponse])
async def get_notebooks(
    archived: Optional[bool] = Query(None, description="Filter by archived status"),
    order_by: str = Query("updated desc", description="Order by field and direction"),
):
    """Get all notebooks with optional filtering and ordering."""
    # Implementation
```

### Request/Response 模型

使用 Pydantic 模型進行驗證:

```python
from pydantic import BaseModel, Field
from typing import Optional

class NotebookCreate(BaseModel):
    name: str = Field(..., description="Name of the notebook", min_length=1)
    description: str = Field(default="", description="Description of the notebook")

class NotebookResponse(BaseModel):
    id: str
    name: str
    description: str
    archived: bool
    created: str
    updated: str
```

### 錯誤處理

使用一致的錯誤回應:

```python
from fastapi import HTTPException
from loguru import logger

try:
    result = await some_operation()
    return result
except InvalidInputError as e:
    raise HTTPException(status_code=400, detail=str(e))
except DatabaseOperationError as e:
    logger.error(f"Database error: {str(e)}")
    raise HTTPException(status_code=500, detail="Internal server error")
```

### API 文件

使用 FastAPI 的自動文件功能:

```python
@router.post(
    "/notebooks",
    response_model=NotebookResponse,
    summary="Create a new notebook",
    description="Create a new notebook with the specified name and description.",
    responses={
        201: {"description": "Notebook created successfully"},
        400: {"description": "Invalid input data"},
        500: {"description": "Internal server error"}
    }
)
async def create_notebook(notebook: NotebookCreate):
    """Create a new notebook."""
    # Implementation
```

## 資料庫標準

### SurrealDB 模式

一致地使用 repository 模式:

```python
from open_notebook.database.repository import repo_create, repo_query, repo_update

# Create records
async def create_notebook(data: Dict[str, Any]) -> Dict[str, Any]:
    """Create a new notebook record."""
    return await repo_create("notebook", data)

# Query with parameters
async def find_notebooks_by_user(user_id: str) -> List[Dict[str, Any]]:
    """Find notebooks for a specific user."""
    return await repo_query(
        "SELECT * FROM notebook WHERE user_id = $user_id",
        {"user_id": user_id}
    )

# Update records
async def update_notebook(notebook_id: str, data: Dict[str, Any]) -> Dict[str, Any]:
    """Update a notebook record."""
    return await repo_update("notebook", notebook_id, data)
```

### 結構描述(schema)管理

結構變更請使用遷移:

```surrealql
-- migrations/8.surrealql
DEFINE TABLE IF NOT EXISTS new_feature SCHEMAFULL;
DEFINE FIELD IF NOT EXISTS name ON TABLE new_feature TYPE string;
DEFINE FIELD IF NOT EXISTS description ON TABLE new_feature TYPE option<string>;
DEFINE FIELD IF NOT EXISTS created ON TABLE new_feature TYPE datetime DEFAULT time::now();
DEFINE FIELD IF NOT EXISTS updated ON TABLE new_feature TYPE datetime DEFAULT time::now();
```

## TypeScript 標準

### 基本準則

遵循 TypeScript 最佳實踐:

- 在 `tsconfig.json` 中啟用嚴格模式
- 所有變數與函式都使用正確的型別註記
- 避免使用 `any` 型別,除非絕對必要
- 物件形狀使用 `interface`,聯集型別與其他進階型別使用 `type`

### 元件結構

- 使用具備 hooks 的函式元件
- 保持元件聚焦且單一職責
- 將可重用邏輯抽取成自訂 hooks
- props 使用正確的 TypeScript 型別

### 錯誤處理

- 明確處理錯誤
- 提供有意義的錯誤訊息
- 適當記錄錯誤
- 不要靜默壓制錯誤

## 程式碼品質工具

我們使用以下工具維持程式碼品質:

- **Ruff**:Lint 與程式碼格式化
  - 執行方式:`uv run ruff check . --fix`
  - 格式化方式:`uv run ruff format .`

- **MyPy**:靜態型別檢查
  - 執行方式:`uv run python -m mypy .`

- **Pytest**:測試框架
  - 執行方式:`uv run pytest`

## 常見模式

### 非同步資料庫操作

```python
async def get_notebook_with_sources(notebook_id: str) -> Notebook:
    """Retrieve notebook with all related sources."""
    notebook_data = await repo_query(
        "SELECT * FROM notebook WHERE id = $id",
        {"id": notebook_id}
    )
    if not notebook_data:
        raise InvalidInputError(f"Notebook {notebook_id} not found")

    sources_data = await repo_query(
        "SELECT * FROM source WHERE notebook_id = $notebook_id",
        {"notebook_id": notebook_id}
    )

    return Notebook(
        **notebook_data[0],
        sources=[Source(**s) for s in sources_data]
    )
```

### 模型驗證

```python
from pydantic import BaseModel, validator

class NotebookInput(BaseModel):
    name: str
    description: str = ""

    @validator('name')
    def name_not_empty(cls, v):
        if not v.strip():
            raise ValueError('Name cannot be empty')
        return v.strip()
```

## 程式碼審查檢查清單

提交程式碼審查前,請確認:

- [ ] 程式碼符合 PEP 8 / TypeScript 最佳實踐
- [ ] 所有函式都有型別提示
- [ ] Docstring 完整且準確
- [ ] 錯誤處理恰當
- [ ] 已包含測試且測試通過
- [ ] 沒有殘留除錯程式碼(console.log、print 陳述式)
- [ ] Commit 訊息清楚且符合慣例
- [ ] 如有需要,文件已更新

---

**另請參閱:**
- [測試指南](testing.zh.md) - 如何撰寫測試
- [貢獻指南](contributing.zh.md) - 整體貢獻工作流程
