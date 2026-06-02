# oryx-re-docs

Oryx-RE 开发文档站，基于 [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)，部署在 GitHub Pages：

👉 https://breakices.github.io/oryx-re-docs/

## 本地预览

```bash
python -m venv .venv && source .venv/Scripts/activate   # Windows Git Bash
# 或 PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements-docs.txt
mkdocs serve            # http://127.0.0.1:8000
```

## 写文档

所有页面在 `docs/` 下，纯 Markdown。新增页面后到 `mkdocs.yml` 的 `nav:` 里挂一条即可。

## 发布

push 到 `main` 分支即可，`.github/workflows/docs.yml` 会自动 `mkdocs build` 并发布到 Pages。

> 首次使用需在仓库 **Settings → Pages → Build and deployment → Source** 选择 **GitHub Actions**。
