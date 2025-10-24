Set Up Via GPT

"""
setup_env.py
------------
Creates a new GitHub repository and a matching local Obsidian Vault folder.

Usage:
    python setup_env.py --repo "sophies-universum" --vault "Sophies Vault"

Requirements:
    pip install PyGithub
    export GITHUB_TOKEN="<your_personal_access_token>"
"""

import os
import argparse
from github import Github
from pathlib import Path
import subprocess

def create_github_repo(repo_name, private=True, description=""):
    """Create a new GitHub repository."""
    token = os.getenv("GITHUB_TOKEN")
    if not token:
        raise EnvironmentError("Please set your GITHUB_TOKEN environment variable.")

    g = Github(token)
    user = g.get_user()

    repo = user.create_repo(
        name=repo_name,
        private=private,
        description=description or f"Auto-created repo for {repo_name}"
    )
    print(f"✅ GitHub repo created: {repo.html_url}")
    return repo

def create_obsidian_vault(vault_name, repo_url=None):
    """Create local Obsidian Vault and optionally initialize Git repo."""
    vault_path = Path.home() / "Obsidian" / vault_name
    vault_path.mkdir(parents=True, exist_ok=True)

    print(f"📂 Obsidian vault created at: {vault_path}")

    # Initialize Git repo locally
    subprocess.run(["git", "init"], cwd=vault_path)
    subprocess.run(["git", "add", "."], cwd=vault_path)
    subprocess.run(["git", "commit", "-m", "Initial vault commit"], cwd=vault_path)

    if repo_url:
        subprocess.run(["git", "remote", "add", "origin", repo_url], cwd=vault_path)
        print(f"🔗 Linked local vault to GitHub: {repo_url}")

    return vault_path

def main():
    parser = argparse.ArgumentParser(description="Setup GitHub repo and Obsidian vault.")
    parser.add_argument("--repo", required=True, help="GitHub repository name")
    parser.add_argument("--vault", required=True, help="Obsidian vault name")
    parser.add_argument("--public", action="store_true", help="Make GitHub repo public")
    parser.add_argument("--desc", default="", help="GitHub repo description")
    args = parser.parse_args()

    repo = create_github_repo(args.repo, private=not args.public, description=args.desc)
    vault_path = create_obsidian_vault(args.vault, repo.clone_url)

    print("\n🎉 Setup complete!")
    print(f"Vault: {vault_path}")
    print(f"GitHub: {repo.html_url}")

if __name__ == "__main__":
    main()

---

🧠 How It Works
You run the script from a terminal:

python setup_env.py --repo "sophies-universum" --vault "Sophies Universum Vault" --desc "AI Vocabulary Trainer integrated with Telegram"
It:

Uses your GitHub token to create a repo.

Creates a folder under ~/Obsidian/ for your vault.

Initializes it as a Git repo and links it to GitHub.

🧩 Requirements
Install dependencies:

pip install PyGithub
and set your token:

export GITHUB_TOKEN="ghp_YourAccessTokenHere"
Would you like me to extend it so it automatically pushes the initial commit to GitHub and creates an Obsidian .obsidian config folder with a vault name?
