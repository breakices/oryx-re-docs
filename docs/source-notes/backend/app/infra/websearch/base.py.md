# `backend/app/infra/websearch/base.py`

> 145 行 · 12 个符号　|　[源码导读对应模块](../source-guide/index.md)

**整体作用**：

<!-- 一句话写这个文件干嘛的 -->

## 符号清单（对照手写）

- `class ProviderError(Exception)` · L26
- `class ProviderRateLimitError(ProviderError)` · L30
- `class ProviderConfigError(ProviderError)` · L34
- `class ProviderTransientError(ProviderError)` · L39
- `class SearchResult` · L44
- `class SearchResponse` · L54
- `class ExtractedPage` · L64
- `class SearchProvider(ABC)` · L72
    - `def search()` · L76
    - `def extract()` · L92
    - `def supports_extract()` · L105
    - `def search_and_extract()` · L108

<!-- 写法建议：在某个符号下另起一行缩进写它的职责/入参/副作用/坑 -->

