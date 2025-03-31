# Managing Multiple GitHub Accounts via SSH

## Scenario

Imagine you are working at a company that already has its GitHub access configured on your laptop. Your company has provided you with an SSH key, and all your work projects authenticate smoothly with GitHub.

Now, you decide to contribute to open-source projects or maintain personal repositories using your **private GitHub account**.  
But when you try to push to your personal repositories, you get this frustrating error:

```
ERROR: Permission to md-anbari/your-repo.git denied to mohammad-anbari-maersk.
```

This happens because:
- Your terminal is still trying to use your **company's SSH key** for your personal repositories.
- GitHub can't guess which account you are intending to use, it just looks at the key you send.

---

## The Goal

We want to make sure:
- Company repositories continue to work without change.
- Personal repositories automatically use your personal key.
- We don't manually switch keys every time.

---

## Pre-requisite

Generate a second key for your personal GitHub if you haven't:

```bash
ssh-keygen -t rsa -b 4096 -C "your_personal_email@example.com"
# Save as ~/.ssh/id_rsa_personal
```

Add the **public key** to your **personal GitHub** account under:  
GitHub → Settings → SSH and GPG keys → New SSH key

---

# Solution 1: The Host Alias Approach

In this classic approach, you modify `~/.ssh/config` to introduce a **host alias**.

```plaintext
# Company GitHub Account
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_company

# Personal GitHub Account (alias)
Host github.com-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_personal
```

### How it works:
When cloning your personal repositories, you now have to use:

```bash
git clone git@github.com-personal:your-username/your-repo.git
```

Your work repositories continue to use:

```bash
git clone git@github.com:your-company/repo.git
```

This way, SSH will pick the correct key automatically based on the **host alias**.

### Drawback:
You must remember to use the custom alias `github.com-personal` whenever you clone or update remotes of personal repositories.

---

# Solution 2: Using `GIT_SSH_COMMAND`

This is a more dynamic solution where you **do not change the remote URL**, but instead specify which SSH key to use **per command**.

Example:

```bash
GIT_SSH_COMMAND='ssh -i ~/.ssh/id_rsa_personal -o IdentitiesOnly=yes' git clone git@github.com:your-username/your-repo.git
```

Or when pushing:

```bash
GIT_SSH_COMMAND='ssh -i ~/.ssh/id_rsa_personal -o IdentitiesOnly=yes' git push
```

### Advantage:
- You can keep using `git@github.com:your-username/repo.git`.
- No need to change the remote URL.

### Drawback:
- You need to **remember** to provide the `GIT_SSH_COMMAND` every time you interact with the repository unless you wrap it in a shell script or alias.
- Might not be very convenient for daily use unless you automate it.

---

# Solution 3: Git Conditional Configuration (`core.sshCommand` + `includeIf`)

This is the cleanest if you naturally organize your repositories into folders like:

```
~/git/work/...
~/git/personal/...
```

Then, you can let **Git itself** decide automatically which key and identity to use **without changing the remote URL** or remembering commands.

### Example `~/.gitconfig`

```ini
[includeIf "gitdir:~/git/work/"]
    path = ~/.gitconfig.work

[includeIf "gitdir:~/git/personal/"]
    path = ~/.gitconfig.personal
```

### Example `~/.gitconfig.personal`

```ini
[user]
    name = Your Personal Name
    email = your.personal@email.com

[core]
    sshCommand = "ssh -i ~/.ssh/id_rsa_personal -o IdentitiesOnly=yes"
```

### Example `~/.gitconfig.work`

```ini
[user]
    name = Your Company Name
    email = your@company.com

[core]
    sshCommand = "ssh -i ~/.ssh/id_rsa_company -o IdentitiesOnly=yes"
```

### How it works:
- If you run Git inside `~/git/personal/`, Git will use `id_rsa_personal` and your personal Git identity automatically.
- If you run Git inside `~/git/work/`, Git will use your company identity and key automatically.
- You continue to use normal GitHub URLs like:
    ```bash
    git@github.com:your-username/your-repo.git
    ```

### Advantage:
- **No need to change remote URLs**.
- Git automatically selects the correct key based on **directory**.
- You manage both Git identity (name, email) and SSH authentication seamlessly.

### Drawback:
- Requires you to organize your repositories by directory (e.g., `~/git/personal` vs `~/git/work`).
- Slightly more advanced Git config.
