# 实验室离线部署镜像：仅包含运行依赖，无外部网络请求
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# 构建前置检查：构建上下文必须包含应用运行所需的全部文件。
# 缺少任何一个必需文件都让构建立即失败，绝不产出“能启动却缺资源”的镜像。
COPY requirements.txt ./
COPY app ./app
COPY static ./static
RUN set -eux; \
    for f in requirements.txt \
             app/__init__.py app/main.py app/bed.py app/chain.py app/engine.py app/report.py \
             static/index.html static/app.js static/style.css; do \
        test -f "$f" || { echo "[build] 缺少构建必需文件: $f" >&2; exit 1; }; \
    done

# 安装运行依赖
RUN pip install --no-cache-dir -r requirements.txt

# 安装后自检：运行依赖可导入、应用入口可加载。
# 依赖缺失或入口损坏时构建失败，而不是让容器带病启动后才在健康检查中暴露。
RUN python -c "import fastapi, uvicorn, multipart; import app.main; assert app.main.app"

# 非 root 用户运行
RUN useradd --create-home --uid 10001 appuser \
    && chown -R appuser:appuser /app
USER appuser

EXPOSE 8080

# 容器内固定监听 8080；宿主机端口映射通过 compose / docker -p 配置
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8080/healthz', timeout=4).status==200 else 1)"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
