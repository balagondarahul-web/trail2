from pathlib import Path
from typing import List, Dict, Optional
import yaml


# ── Stage Templates ────────────────────────────────────────────────────────────

def stage(name: str, commands: List[str], **kwargs) -> Dict:
    """Generic stage builder."""
    base = {
        "name": name,
        "commands": commands,
    }
    base.update(kwargs)
    return base


def build_stage(image: str = "python:3.11") -> Dict:
    return stage(
        name="build",
        image=image,
        commands=[
            "pip install --upgrade pip",
            "pip install -r requirements.txt",
            "python -m py_compile src/*.py",
        ],
    )


def test_stage(image: str = "python:3.11") -> Dict:
    return stage(
        name="test",
        image=image,
        commands=[
            "pip install -r requirements.txt",
            "pytest tests/ -v --tb=short",
            "coverage run -m pytest tests/",
            "coverage report --fail-under=80",
        ],
    )


def deploy_stage(env: str = "production") -> Dict:
    return stage(
        name="deploy",
        environment=env,
        when="on_success",
        commands=[
            f"echo 'Deploying to {env}...'",
            "scp -r ./dist user@server:/var/www/app",
            "ssh user@server 'systemctl restart myapp'",
        ],
    )


def notify_stage(condition: str = "always") -> Dict:
    return stage(
        name="notify",
        when=condition,
        commands=[
            "echo 'Pipeline complete. Status: $CI_JOB_STATUS'",
            "curl -X POST $SLACK_WEBHOOK -d '{\"text\":\"Deploy done\"}'",
        ],
    )


# ── GitHub Actions ─────────────────────────────────────────────────────────────

def create_github_pipeline(
    name: str = "CI/CD Pipeline",
    python_version: str = "3.11",
    branches: Optional[List[str]] = None,
    deploy_env: str = "production",
) -> Dict:
    branches = branches or ["main", "develop"]

    def setup_python():
        return {
            "name": "Set up Python",
            "uses": "actions/setup-python@v5",
            "with": {"python-version": python_version},
        }

    def install_deps():
        return {
            "name": "Install dependencies",
            "run": "pip install --upgrade pip\npip install -r requirements.txt",
        }

    return {
        "name": name,
        "on": {
            "push": {"branches": branches},
            "pull_request": {"branches": branches},
        },
        "jobs": {
            "build": {
                "runs-on": "ubuntu-latest",
                "steps": [
                    {"uses": "actions/checkout@v4"},
                    setup_python(),
                    install_deps(),
                    {
                        "name": "Compile check",
                        "run": "python -m py_compile $(find src -name '*.py')",
                    },
                ],
            },
            "test": {
                "runs-on": "ubuntu-latest",
                "needs": "build",
                "steps": [
                    {"uses": "actions/checkout@v4"},
                    setup_python(),
                    install_deps(),
                    {"name": "Run tests", "run": "pytest tests/ -v --tb=short"},
                    {
                        "name": "Coverage",
                        "run": "coverage run -m pytest tests/\ncoverage report --fail-under=80",
                    },
                ],
            },
            "deploy": {
                "runs-on": "ubuntu-latest",
                "needs": "test",
                "if": "github.ref == 'refs/heads/main'",
                "environment": deploy_env,
                "steps": [
                    {"uses": "actions/checkout@v4"},
                    {
                        "name": f"Deploy to {deploy_env}",
                        "run": f"echo 'Deploying to {deploy_env}...'\n# deployment here",
                        "env": {
                            "DEPLOY_KEY": "${{ secrets.DEPLOY_KEY }}",
                            "SERVER_HOST": "${{ secrets.SERVER_HOST }}",
                        },
                    },
                ],
            },
        },
    }


# ── GitLab CI ─────────────────────────────────────────────────────────────────

def create_gitlab_pipeline(
    image: str = "python:3.11",
    deploy_env: str = "production",
) -> Dict:
    return {
        "image": image,
        "stages": ["build", "test", "deploy", "notify"],
        "variables": {
            "PIP_CACHE_DIR": "$CI_PROJECT_DIR/.cache/pip",
        },
        "cache": {"paths": [".cache/pip", "venv/"]},

        "build": {
            "stage": "build",
            "script": [
                "pip install --upgrade pip",
                "pip install -r requirements.txt",
                "python -m py_compile $(find src -name '*.py')",
            ],
            "artifacts": {"paths": ["dist/"], "expire_in": "1 hour"},
        },

        "test": {
            "stage": "test",
            "script": [
                "pip install -r requirements.txt",
                "pytest tests/ -v --tb=short --junitxml=report.xml",
            ],
            "coverage": r"/TOTAL.*\s+(\d+%)/",
            "artifacts": {"reports": {"junit": "report.xml"}},
        },

        "lint": {
            "stage": "test",
            "script": [
                "pip install flake8 black",
                "flake8 src/ --max-line-length=100",
                "black src/ --check",
            ],
        },

        "deploy": {
            "stage": "deploy",
            "script": [
                f"echo 'Deploying to {deploy_env}'",
                "# deployment script",
            ],
            "environment": {
                "name": deploy_env,
                "url": f"https://{deploy_env}.example.com",
            },
            "only": ["main"],
        },

        "notify": {
            "stage": "notify",
            "when": "always",
            "script": [
                'curl -X POST "$SLACK_WEBHOOK" '
                '-H "Content-Type: application/json" '
                '-d \'{"text": "Pipeline $CI_PIPELINE_STATUS"}\'',
            ],
        },
    }


# ── Custom Pipeline ────────────────────────────────────────────────────────────

def create_custom_pipeline(stages: Optional[List[Dict]] = None) -> Dict:
    return {
        "version": "1.0",
        "pipeline": {
            "name": "Custom CI/CD Pipeline",
            "stages": stages or [
                build_stage(),
                test_stage(),
                deploy_stage(),
                notify_stage(),
            ],
        },
    }


# ── File Writer ────────────────────────────────────────────────────────────────

def save_pipeline(data: Dict, filepath: str) -> None:
    path = Path(filepath)
    path.parent.mkdir(parents=True, exist_ok=True)

    with path.open("w", encoding="utf-8") as f:
        yaml.dump(data, f, sort_keys=False, allow_unicode=True)

    print(f"✔ Saved: {path}")


# ── Main Execution ─────────────────────────────────────────────────────────────

if __name__ == "__main__":
    save_pipeline(
        create_github_pipeline(name="My App CI/CD"),
        ".github/workflows/pipeline.yml",
    )

    save_pipeline(
        create_gitlab_pipeline(deploy_env="staging"),
        ".gitlab-ci.yml",
    )

    save_pipeline(
        create_custom_pipeline(),
        "pipeline.yml",
    )

    print("\n✅ All pipelines generated successfully.")
