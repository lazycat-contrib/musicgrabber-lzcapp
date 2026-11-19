# MusicGrabber LPK 构建镜像：基于上游 Dockerfile，仅调整入口适配懒猫网关。
# 上游 Dockerfile 多阶段构建（Bento4 编译 + Python 运行时 + Playwright/SeleniumBase 浏览器栈）。
# 构建上下文 = ./upstream（build.sh 拉取的固定 commit 源码），本文件中 COPY 均相对 upstream/。
# 说明：upstream/Dockerfile 的 HEALTHCHECK 引用容器内 localhost:8080，保留原样可用。

FROM python:3.12-slim AS bento4-build
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl cmake g++ make && rm -rf /var/lib/apt/lists/*
RUN curl -fSL https://codeload.github.com/axiomatic-systems/Bento4/tar.gz/dc264854d1f76c370b65b18d9f303a95f7f21ab1 -o /tmp/bento4.tar.gz && \
    echo "756ebf50c60d674027ef815d3b961dc01688ac270a04636144568192f04528c9  /tmp/bento4.tar.gz" | sha256sum -c - && \
    mkdir /tmp/bento4 && tar -xzf /tmp/bento4.tar.gz -C /tmp/bento4 --strip-components=1 && \
    cmake -S /tmp/bento4 -B /tmp/bento4-build -DCMAKE_BUILD_TYPE=Release && \
    cmake --build /tmp/bento4-build --target mp4decrypt --parallel 2 && \
    strip /tmp/bento4-build/mp4decrypt

FROM python:3.12-slim

RUN apt-get update && apt-get upgrade -y && apt-get install -y --no-install-recommends \
    ffmpeg \
    curl \
    gosu \
    libchromaprint-tools \
    tini \
    && rm -rf /var/lib/apt/lists/*

RUN ARCH=$(dpkg --print-architecture) && \
    if [ "$ARCH" = "arm64" ]; then \
        YT_DLP_BIN="yt-dlp_linux_aarch64"; \
    else \
        YT_DLP_BIN="yt-dlp"; \
    fi && \
    curl -L "https://github.com/yt-dlp/yt-dlp/releases/latest/download/${YT_DLP_BIN}" \
        -o /usr/local/bin/yt-dlp && \
    chmod a+rx /usr/local/bin/yt-dlp

COPY --from=denoland/deno:bin-2.9.7 /deno /usr/local/bin/deno

RUN pip install --no-cache-dir --upgrade pip setuptools wheel && \
    pip install --no-cache-dir \
    fastapi~=0.162.0 \
    uvicorn[standard]~=0.54.0 \
    httpx~=0.28.1 \
    pydantic~=2.14.1 \
    mutagen~=1.48.1 \
    playwright~=1.64.1 \
    "apprise>=1.13.1,<2" \
    bcrypt~=4.3.0 \
    pycryptodome~=4.2.0 \
    "certifi>=2026.10.18"

RUN pip install --no-cache-dir "seleniumbase>=4.54,<5"

ENV PLAYWRIGHT_BROWSERS_PATH=/ms-playwright
RUN playwright install chromium --with-deps

RUN apt-get update && apt-get install -y --no-install-recommends xauth tk8.6 && \
    rm -rf /var/lib/apt/lists/*

RUN ARCH=$(dpkg --print-architecture) && \
    if [ "$ARCH" = "amd64" ]; then \
        curl -L https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \
            -o /tmp/google-chrome.deb && \
        apt-get update && apt-get install -y --no-install-recommends /tmp/google-chrome.deb && \
        rm -f /tmp/google-chrome.deb; \
    else \
        apt-get update && apt-get install -y --no-install-recommends chromium chromium-driver; \
    fi && \
    rm -rf /var/lib/apt/lists/*

RUN ARCH=$(dpkg --print-architecture) && \
    if [ "$ARCH" = "amd64" ]; then \
        sbase get uc_driver stable; \
    else \
        cp /usr/bin/chromedriver \
            /usr/local/lib/python3.12/site-packages/seleniumbase/drivers/uc_driver && \
        chmod 755 /usr/local/lib/python3.12/site-packages/seleniumbase/drivers/uc_driver; \
    fi

RUN python -c "from pathlib import Path; import seleniumbase; from seleniumbase.undetected.patcher import Patcher; driver = Path(seleniumbase.__file__).parent / 'drivers' / 'uc_driver'; Patcher(executable_path=str(driver)).auto(); driver.chmod(0o755)"

RUN find /usr/local/lib/python3.12/site-packages/selenium/webdriver/common -name 'selenium-manager*' -delete

COPY --from=bento4-build /tmp/bento4-build/mp4decrypt /usr/local/bin/mp4decrypt
RUN python3 -m venv /opt/soundcloud-drm && \
    /opt/soundcloud-drm/bin/pip install --no-cache-dir --upgrade pip setuptools wheel && \
    curl -fSL https://codeload.github.com/arabikCC/yt-dlp-soundcloud-drm/tar.gz/7441f905b3b39d3314b3fc7d49cb29fd34e74095 -o /tmp/soundcloud-plugin.tar.gz && \
    echo "9ac2bdf4a586f2729eaee159f3273c21ab444c92a5632a1e4097e7f87155a2cf  /tmp/soundcloud-plugin.tar.gz" | sha256sum -c - && \
    /opt/soundcloud-drm/bin/pip install --no-cache-dir \
      'yt-dlp==2026.10.26' 'pywidevine==1.9.0' 'curl-cffi==0.17.0' /tmp/soundcloud-plugin.tar.gz && \
    /opt/soundcloud-drm/bin/pip uninstall -y pip && \
    rm /tmp/soundcloud-plugin.tar.gz

RUN python3 -m venv /opt/flac-verifier && \
    /opt/flac-verifier/bin/pip install --no-cache-dir \
      'numpy==2.2.6' 'scipy==1.15.3' 'soundfile==0.13.1' 'mutagen==1.47.0' && \
    curl -fSL https://codeload.github.com/FraysKosher/flac-verifier/tar.gz/9e17f9f787fca58803f753fd9be06179efd4096e -o /tmp/flac-verifier.tar.gz && \
    echo "e145a98c5a7170ba8237eb7ce6aa76e8a20f51c15bf1366c106ba8e838df1f67  /tmp/flac-verifier.tar.gz" | sha256sum -c - && \
    mkdir -p /opt/flac-verifier/engine && \
    tar -xzf /tmp/flac-verifier.tar.gz -C /opt/flac-verifier/engine --strip-components=1 \
      --wildcards '*/motor_flac.py' '*/_motor_*.py' '*/LICENSE' && \
    /opt/flac-verifier/bin/pip uninstall -y pip && \
    rm /tmp/flac-verifier.tar.gz

RUN pip uninstall -y pip

RUN mkdir -p /app
WORKDIR /opt/musicgrabber

COPY upstream/*.py /opt/musicgrabber/
COPY upstream/core /opt/musicgrabber/core/
COPY upstream/toolkit /opt/musicgrabber/toolkit/
COPY upstream/sources /opt/musicgrabber/sources/
COPY upstream/search /opt/musicgrabber/search/
COPY upstream/platforms /opt/musicgrabber/platforms/
COPY upstream/library /opt/musicgrabber/library/
COPY upstream/ingest /opt/musicgrabber/ingest/
COPY upstream/metadata /opt/musicgrabber/metadata/
COPY upstream/downloads /opt/musicgrabber/downloads/
COPY upstream/routes /opt/musicgrabber/routes/
COPY upstream/static /opt/musicgrabber/static/
COPY upstream/entrypoint.sh /opt/musicgrabber/

RUN mkdir -p /data /music && chmod 777 /music

RUN chmod +x /opt/musicgrabber/entrypoint.sh

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:${LISTEN_PORT:-8080}/ || exit 1

ENV PYTHONUNBUFFERED=1

ENTRYPOINT ["/usr/bin/tini", "--"]

CMD ["/opt/musicgrabber/entrypoint.sh"]
