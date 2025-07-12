# Gensyn
## Gensyn  rl-swarm folder edit for 3090 Qwen/Qwen3-0.6B  (model)

``` #!/bin/bash

set -euo pipefail

# General arguments
ROOT=$PWD

# GenRL Swarm version to use
GENRL_TAG="v0.1.1"

export IDENTITY_PATH
export GENSYN_RESET_CONFIG
export CONNECT_TO_TESTNET=true
export ORG_ID
export HF_HUB_DOWNLOAD_TIMEOUT=120
export SWARM_CONTRACT="0xFaD7C5e93f28257429569B854151A1B8DCD404c2"
export HUGGINGFACE_ACCESS_TOKEN="None"

DEFAULT_IDENTITY_PATH="$ROOT"/swarm.pem
IDENTITY_PATH=${IDENTITY_PATH:-$DEFAULT_IDENTITY_PATH}
DOCKER=${DOCKER:-""}
GENSYN_RESET_CONFIG=${GENSYN_RESET_CONFIG:-""}
CPU_ONLY=${CPU_ONLY:-""}
ORG_ID=${ORG_ID:-""}

GREEN_TEXT="\033[32m"
BLUE_TEXT="\033[34m"
RED_TEXT="\033[31m"
RESET_TEXT="\033[0m"

echo_green() { echo -e "$GREEN_TEXT$1$RESET_TEXT"; }
echo_blue()  { echo -e "$BLUE_TEXT$1$RESET_TEXT"; }
echo_red()   { echo -e "$RED_TEXT$1$RESET_TEXT"; }

ROOT_DIR="$(cd $(dirname ${BASH_SOURCE[0]}) && pwd)"

cleanup() {
    echo_green ">> Shutting down trainer..."
    # Don’t delete login session
    # rm -r $ROOT_DIR/modal-login/temp-data/*.json 2> /dev/null || true
    kill -- -$$ || true
    exit 0
}

errnotify() {
    echo_red ">> An error was detected while running rl-swarm. See $ROOT/logs for full logs."
}

trap cleanup EXIT
trap errnotify ERR

echo -e "\033[38;5;224m"
cat << "EOF"
    ██████  ██            ███████ ██     ██  █████  ██████  ███    ███
    ██   ██ ██            ██      ██     ██ ██   ██ ██   ██ ████  ████
    ██████  ██      █████ ███████ ██  █  ██ ███████ ██████  ██ ████ ██
    ██   ██ ██                 ██ ██ ███ ██ ██   ██ ██   ██ ██  ██  ██
    ██   ██ ███████       ███████  ███ ███  ██   ██ ██   ██ ██      ██

    From Gensyn
EOF

mkdir -p "$ROOT/logs"

if [ "$CONNECT_TO_TESTNET" = true ]; then
    if [ -f "modal-login/temp-data/userData.json" ]; then
        echo_green ">> Found existing login data. Skipping login..."

        ORG_ID=$(awk 'BEGIN { FS = "\"" } /orgId/ { print $(NF - 1); exit }' modal-login/temp-data/userData.json)
        echo "Your ORG_ID is set to: $ORG_ID"

        # Start modal-login server if not running
        if ! nc -z localhost 3000; then
            echo ">> Starting modal-login server manually..."
            cd modal-login
            yarn start >> "$ROOT/logs/yarn.log" 2>&1 &
            cd ..
            sleep 5
        fi

        echo "Waiting for API key to become activated..."
        while true; do
            STATUS=$(curl -s "http://localhost:3000/api/get-api-key-status?orgId=$ORG_ID")
            if [ -z "$STATUS" ]; then
                echo_red ">> Error: Failed to connect to local judge server at http://localhost:3000"
                exit 1
            fi
            if [[ "$STATUS" == "activated" ]]; then
                echo "API key is activated! Proceeding..."
                break
            else
                echo "Waiting for API key to be activated..."
                sleep 5
            fi
        done

    else
        echo "Please login to create an Ethereum Server Wallet"
        cd modal-login

        if ! command -v node > /dev/null 2>&1; then
            echo "Node.js not found. Installing NVM and latest Node.js..."
            export NVM_DIR="$HOME/.nvm"
            if [ ! -d "$NVM_DIR" ]; then
                curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
            fi
            [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
            [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
            nvm install node
        else
            echo "Node.js is already installed: $(node -v)"
        fi

        if ! command -v yarn > /dev/null 2>&1; then
            if grep -qi "ubuntu" /etc/os-release 2> /dev/null || uname -r | grep -qi "microsoft"; then
                echo "Detected Ubuntu or WSL Ubuntu. Installing Yarn via apt..."
                curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
                echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
                sudo apt update && sudo apt install -y yarn
            else
                npm install -g --silent yarn
            fi
        fi

        ENV_FILE="$ROOT/modal-login/.env"
        if [[ "$OSTYPE" == "darwin"* ]]; then
            sed -i '' "3s/.*/SMART_CONTRACT_ADDRESS=$SWARM_CONTRACT/" "$ENV_FILE"
        else
            sed -i "3s/.*/SMART_CONTRACT_ADDRESS=$SWARM_CONTRACT/" "$ENV_FILE"
        fi

        if [ -z "$DOCKER" ]; then
            yarn install --immutable
            echo "Building server"
            yarn build > "$ROOT/logs/yarn.log" 2>&1
        fi

        yarn start >> "$ROOT/logs/yarn.log" 2>&1 &

        cd ..

        echo_green ">> Waiting for modal userData.json to be created..."
        while [ ! -f "modal-login/temp-data/userData.json" ]; do
            sleep 5
        done
        echo "Found userData.json. Proceeding..."

        ORG_ID=$(awk 'BEGIN { FS = "\"" } /orgId/ { print $(NF - 1); exit }' modal-login/temp-data/userData.json)
        echo "Your ORG_ID is set to: $ORG_ID"

        echo "Waiting for API key to become activated..."
        while true; do
            STATUS=$(curl -s "http://localhost:3000/api/get-api-key-status?orgId=$ORG_ID")
            if [[ "$STATUS" == "activated" ]]; then
                echo "API key is activated! Proceeding..."
                break
            else
                echo "Waiting for API key to be activated..."
                sleep 5
            fi
        done
    fi
fi

echo_green ">> Getting requirements..."
pip install --upgrade pip
pip install gensyn-genrl==0.1.4
pip install reasoning-gym>=0.1.20
pip install trl
pip install hivemind@git+https://github.com/learning-at-home/hivemind@4d5c41495be082490ea44cce4e9dd58f9926bb4e

if [ ! -d "$ROOT/configs" ]; then mkdir "$ROOT/configs"; fi

if [ -f "$ROOT/configs/rg-swarm.yaml" ]; then
    if ! cmp -s "$ROOT/rgym_exp/config/rg-swarm.yaml" "$ROOT/configs/rg-swarm.yaml"; then
        if [ -z "$GENSYN_RESET_CONFIG" ]; then
            echo_green ">> Found differences in rg-swarm.yaml. To reset, set GENSYN_RESET_CONFIG."
        else
            echo_green ">> Backing up and replacing rg-swarm.yaml"
            mv "$ROOT/configs/rg-swarm.yaml" "$ROOT/configs/rg-swarm.yaml.bak"
            cp "$ROOT/rgym_exp/config/rg-swarm.yaml" "$ROOT/configs/rg-swarm.yaml"
        fi
    fi
else
    cp "$ROOT/rgym_exp/config/rg-swarm.yaml" "$ROOT/configs/rg-swarm.yaml"
fi

if [ -n "$DOCKER" ]; then
    sudo chmod -R 0777 /home/gensyn/rl_swarm/configs
fi

echo_green ">> Done!"

HF_TOKEN=${HF_TOKEN:-""}
if [ -n "${HF_TOKEN}" ]; then
    HUGGINGFACE_ACCESS_TOKEN=${HF_TOKEN}
else
    echo -en $GREEN_TEXT
    read -p ">> Would you like to push models to Hugging Face Hub? [y/N] " yn
    echo -en $RESET_TEXT
    yn=${yn:-N}
    case $yn in
        [Yy]*) read -p "Enter your Hugging Face access token: " HUGGINGFACE_ACCESS_TOKEN ;;
        [Nn]*) HUGGINGFACE_ACCESS_TOKEN="None" ;;
        *) echo ">>> No answer. Models will not be pushed." && HUGGINGFACE_ACCESS_TOKEN="None" ;;
    esac
fi

echo -en $GREEN_TEXT
read -p ">> Enter the model to use (huggingface repo/name) or press [Enter] for default: " MODEL_NAME
echo -en $RESET_TEXT

if [ -n "$MODEL_NAME" ]; then
    export MODEL_NAME
    echo_green ">> Using model: $MODEL_NAME"
else
    echo_green ">> Using default model from config"
fi

echo_green ">> Good luck in the swarm!"
echo_blue ">> Remember to star the repo: https://github.com/gensyn-ai/rl-swarm"

python -m rgym_exp.runner.swarm_launcher \
    --config-path "$ROOT/rgym_exp/config" \
    --config-name "rg-swarm.yaml"

wait # Keep script running until Ctrl+C
```


COPY ALL CODE AND PASTE INTO YOUR  rl-swarm FOLDER then restart node with:

```./run_rl_swarm.sh```

## This will solve the problem of your node stopping frequently.
