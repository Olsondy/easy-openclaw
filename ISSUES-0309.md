
```
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/shell-helpers/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh

echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc 

clawdock-start

clawdock-fix-token

clawdock-dashboard

export CLAWDOCK_DIR=/d/Resources/MyWorkspace/chore/AI/openclaw
clawdock-start
clawdock-fix-token
clawdock-dashboard
clawdock-devices
clawdock-approve 34e36af4-f23c-4824-a2c5-d570313a4c7e

docker exec openclaw-olsond2-2-openclaw-gateway-1 node dist/index.js devices list

docker exec openclaw-olsond2-2-openclaw-gateway-1 node dist/index.js devices approve 22a37f51-d84b-414e-839a-62f8564128f5

docker build --build-arg OPENCLAW_INSTALL_BROWSER=1 --build-arg OPENCLAW_EXTENSIONS=feishu -t openclaw:local -f Dockerfile .

docker exec -it $c sh -lc "openclaw config set browser.executablePath /home/node/.cache/ms-playwright/chromium-1208/chrome-linux64/chrome"
docker exec -it $c sh -lc "openclaw config set browser.noSandbox true"
docker restart $c
docker exec -it $c sh -lc "openclaw browser status --json"


docker build \
    --build-arg OPENCLAW_EXTENSIONS="feishu" \
    --build-arg OPENCLAW_VARIANT=slim \
    --build-arg OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg" \
    --build-arg OPENCLAW_INSTALL_BROWSER=1 \
    --build-arg OPENCLAW_INSTALL_DOCKER_CLI=1 \
    -t openclaw:local .

openclaw agents add --workspace /home/node/.openclaw/qwen-workspace qwen-video  QA6DUNSA
```