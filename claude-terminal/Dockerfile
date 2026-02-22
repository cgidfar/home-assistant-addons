ARG BUILD_FROM
FROM ${BUILD_FROM}

# Install system packages, development tools, and Claude native binary dependencies
# libgcc + libstdc++: Required for Claude native binary on Alpine (musl)
# ripgrep: Required because Claude's bundled ripgrep doesn't work on musl
RUN apk add --no-cache \
    bash \
    curl \
    nano \
    nodejs \
    npm \
    python3 \
    py3-pip \
    py3-requests \
    py3-aiohttp \
    py3-yaml \
    py3-beautifulsoup4 \
    git \
    vim \
    wget \
    jq \
    tree \
    tmux \
    yq \
    libgcc \
    libstdc++ \
    ripgrep

# Tell Claude Code to use system ripgrep instead of bundled (required for musl/Alpine)
ENV USE_BUILTIN_RIPGREP=0

# Install Claude Code CLI with cascading fallback:
#   1. Direct binary download from GCS (fast, low memory, checksum verified)
#   2. Official native installer (curl | bash) - upstream recommended method
#   3. npm install (deprecated by Anthropic, but covers AVX/armv7 edge cases)
ARG BUILD_ARCH
RUN set -eux; \
    GCS_BUCKET="https://storage.googleapis.com/claude-code-dist-86c565f3-f756-42ad-8dfa-d59b1c096819/claude-code-releases"; \
    INSTALL_DIR="/root/.local/bin"; \
    INSTALL_SUCCESS=false; \
    \
    # Map Home Assistant BUILD_ARCH to Claude platform string \
    case "${BUILD_ARCH}" in \
        amd64)   CLAUDE_PLATFORM="linux-x64-musl" ;; \
        aarch64) CLAUDE_PLATFORM="linux-arm64-musl" ;; \
        *)       CLAUDE_PLATFORM="" ;; \
    esac; \
    \
    # --- Method 1: Direct binary download from GCS --- \
    # Fastest and most memory-efficient; avoids running "claude install" which causes OOM \
    if [ -n "${CLAUDE_PLATFORM}" ]; then \
        echo ">>> [1/3] Attempting direct binary download for ${BUILD_ARCH} (${CLAUDE_PLATFORM})..."; \
        \
        VERSION=$(wget -qO- "${GCS_BUCKET}/stable") || VERSION=""; \
        \
        if [ -n "${VERSION}" ]; then \
            echo ">>> Claude Code stable version: ${VERSION}"; \
            MANIFEST=$(wget -qO- "${GCS_BUCKET}/${VERSION}/manifest.json") || MANIFEST=""; \
            \
            if [ -n "${MANIFEST}" ]; then \
                EXPECTED_SHA=$(echo "${MANIFEST}" | jq -r ".platforms.\"${CLAUDE_PLATFORM}\".checksum // empty"); \
                \
                if [ -n "${EXPECTED_SHA}" ]; then \
                    mkdir -p "${INSTALL_DIR}"; \
                    \
                    if wget -qO "${INSTALL_DIR}/claude" "${GCS_BUCKET}/${VERSION}/${CLAUDE_PLATFORM}/claude"; then \
                        ACTUAL_SHA=$(sha256sum "${INSTALL_DIR}/claude" | awk '{print $1}'); \
                        \
                        if [ "${ACTUAL_SHA}" = "${EXPECTED_SHA}" ]; then \
                            echo ">>> Checksum verified OK"; \
                            chmod +x "${INSTALL_DIR}/claude"; \
                            \
                            if "${INSTALL_DIR}/claude" --version; then \
                                echo ">>> Direct binary install successful!"; \
                                INSTALL_SUCCESS=true; \
                            else \
                                echo ">>> WARNING: Binary failed to execute (possible AVX requirement)"; \
                                rm -f "${INSTALL_DIR}/claude"; \
                            fi; \
                        else \
                            echo ">>> WARNING: Checksum mismatch, removing binary"; \
                            rm -f "${INSTALL_DIR}/claude"; \
                        fi; \
                    else \
                        echo ">>> WARNING: Binary download failed"; \
                    fi; \
                fi; \
            fi; \
        fi; \
    fi; \
    \
    # --- Method 2: Official native installer --- \
    # Uses "curl | bash" like the upstream repo; may OOM on <4GB RAM systems \
    if [ "${INSTALL_SUCCESS}" = "false" ]; then \
        echo ">>> [2/3] Attempting official native installer..."; \
        mkdir -p "${INSTALL_DIR}"; \
        if curl -fsSL https://claude.ai/install.sh | bash; then \
            if [ -f "${INSTALL_DIR}/claude" ] && "${INSTALL_DIR}/claude" --version; then \
                echo ">>> Native installer successful!"; \
                INSTALL_SUCCESS=true; \
            else \
                echo ">>> WARNING: Native installer completed but binary not working"; \
                rm -f "${INSTALL_DIR}/claude"; \
            fi; \
        else \
            echo ">>> WARNING: Native installer failed"; \
        fi; \
    fi; \
    \
    # --- Method 3: npm fallback (deprecated but broad compatibility) --- \
    # Covers armv7, CPUs without AVX, and any other native binary failures \
    if [ "${INSTALL_SUCCESS}" = "false" ]; then \
        echo ">>> [3/3] Falling back to npm install (deprecated but compatible)..."; \
        npm install -g @anthropic-ai/claude-code; \
        echo ">>> npm install complete"; \
    fi; \
    \
    # Create symlink to /usr/local/bin for PATH availability \
    ln -sf "${INSTALL_DIR}/claude" /usr/local/bin/claude \
        || ln -sf "$(which claude)" /usr/local/bin/claude; \
    \
    # Final verification \
    claude --version

# Create directory for Claude configuration and set working directory
RUN mkdir -p /config/claude-config
WORKDIR /config

# Copy startup script and modular scripts
COPY run.sh /run.sh
COPY scripts/ /opt/scripts/
RUN chmod +x /run.sh \
    && chmod +x /opt/scripts/*.sh

# Use exec form for better signal handling
CMD ["/run.sh"]
