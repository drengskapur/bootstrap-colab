# Bootstrap Google Colab

```python
%matplotlib inline

def bootstrap():
    # @title Bootstrap Google Colab {display-mode:"form"}

    # CONFIGURATION PARAMETERS
    GOOGLE_DRIVE_FOLDER = "my-folder" # @param {type:"string"}
    GitHub = True  # @param {type:"boolean"}
    OpenAI = True  # @param {type:"boolean"}
    HuggingFace = True  # @param {type:"boolean"}
    Kaggle = True  # @param {type:"boolean"}

    # ENSURE: Secrets
    from google.colab import userdata
    SECRETS = [
        ("GH_TOKEN", GitHub), # https://github.com/settings/personal-access-tokens/new
        ("GITHUB_USERNAME", GitHub), # git config --global user.name
        ("GITHUB_EMAIL", GitHub), # git config --global user.email
        ("OPENAI_API_KEY", OpenAI), # https://platform.openai.com/api-keys
        ("HF_TOKEN", HuggingFace), # https://huggingface.co/settings/tokens?new_token=true
        ("KAGGLE_USERNAME", Kaggle),
        ("KAGGLE_KEY", Kaggle), # https://www.kaggle.com/settings#:~:text=Create%20New%20Token
    ]
    for secret, enabled in SECRETS:
        if enabled:
            try:
                userdata.get(secret)
            except userdata.SecretNotFoundError:
                raise ValueError(f"Must set Google Colab secret: {secret}.")

    # CONFIGURE: Environment
    import os
    os.environ['PIP_QUIET'] = '3'
    os.environ['PIP_PROGRESS_BAR'] = 'off'
    os.environ['PIP_ROOT_USER_ACTION'] = 'ignore'
    os.environ['DEBIAN_FRONTEND'] = 'noninteractive'

    # CONFIGURE: matplotlib
    import matplotlib.pyplot as plt
    plt.rcParams['figure.dpi'] = 300
    plt.rcParams['savefig.dpi'] = 300

    # DISABLE: Telemetry
    os.environ['HF_HUB_DISABLE_TELEMETRY'] = '1'
    os.environ['GRADIO_ANALYTICS_ENABLED'] = 'False'

    # CONFIGURE: apt
    # https://manpages.ubuntu.com/manpages/bionic/man5/apt.conf.5.html
    APT_CONFIG = [
        'APT::Acquire::Retries "20";',
        'APT::Clean-Installed "true";',
        'APT::Get::Assume-Yes "true";',
        'APT::Get::Clean "always";',
        'APT::Get::Fix-Broken "true";',
        'APT::Install-Recommends "0";',
        'APT::Install-Suggests "0";',
        'APT::Sources::List::Disable-Auto-Refresh "true";',
        'Dpkg::Options "--force-confnew";',
        'Dpkg::Use-Pty "0";',
        'quiet "2";',
    ]
    with open('/etc/apt/apt.conf.d/99apt.conf', 'w') as file:
        for setting in APT_CONFIG:
            file.write(setting + '\n')

    # INSTALL: uv
    # https://github.com/astral-sh/uv
    !pip install uv

    # CONFIGURE: GitHub
    # https://github.com/cli/cli/blob/trunk/docs/install_linux.md
    if GitHub:
        !apt-get remove --purge -y gh > /dev/null
        !mkdir -p -m 755 /etc/apt/keyrings
        !wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null
        !chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg
        !echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null
        !apt-get update > /dev/null
        !apt-get install -y gh > /dev/null
        !gh auth login --hostname "github.com" --git-protocol https --with-token <<< {userdata.get("GH_TOKEN")}
        !git config --global user.name {userdata.get("GITHUB_USERNAME")}
        !git config --global user.email {userdata.get("GITHUB_EMAIL")}

    # AUTHENTICATE: OpenAI
    # https://www.kaggle.com/settings
    if OpenAI:
        os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
        !uv pip install --system --quiet openai
        from openai import OpenAI
        client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))

    # AUTHENTICATE: Hugging Face
    # https://huggingface.co/docs/huggingface_hub/en/quick-start#authentication
    if HuggingFace:
        !uv pip install --system --quiet huggingface_hub[cli]
        !huggingface-cli login --add-to-git-credential --token {userdata.get("HF_TOKEN")} > /dev/null

    # AUTHENTICATE: Kaggle
    # https://www.kaggle.com/settings
    if Kaggle:
        os.environ["KAGGLE_USERNAME"] = userdata.get("KAGGLE_USERNAME")
        os.environ["KAGGLE_KEY"] = userdata.get("KAGGLE_KEY")
        !uv pip install --system --quiet kaggle
        from kaggle.api.kaggle_api_extended import KaggleApi
        api = KaggleApi()
        api.authenticate()

    # MOUNT: Google Drive
    import contextlib
    with contextlib.redirect_stdout(open(os.devnull, 'w')):
        import google.colab
        google.colab.drive.mount("/content/drive", force_remount=True)

    # SYMLINK: Google Drive folder to Files Pane (Top Level)
    import pathlib
    drive_path = pathlib.Path("/content/drive/MyDrive")
    colab_notebooks_path = drive_path / "Colab Notebooks"
    project_path = colab_notebooks_path / GOOGLE_DRIVE_FOLDER
    project_path.mkdir(parents=True, exist_ok=True)
    shortcut = pathlib.Path(f"/content/{GOOGLE_DRIVE_FOLDER}")
    shortcut.parent.mkdir(parents=True, exist_ok=True)
    if not shortcut.exists():
        shortcut.symlink_to(project_path)

    # REMOVE: Sample Folder
    !rm -rf /content/sample_data

    # ENSURE: apt packages
    !apt-get install -qq \
        tree

    # ENSURE: pip packages
    !pip install --upgrade pip
    !uv pip install --system --quiet \
        black[jupyter] \
        isort

    # IMPORT: Python Libraries
    import tqdm.notebook

    # OUTPUTS
    print(f"SHORTCUT: {shortcut} --> {project_path}")
    return str(shortcut)

SHORTCUT = bootstrap()

```
