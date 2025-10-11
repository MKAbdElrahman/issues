
  echo "# issues" >> README.md
  git init
  git add README.md

  # Set user info for personal account
  git config --local user.name "Your Personal Name"
  git config --local user.email "personal@email.com"

  git commit -m "first commit"
  git branch -M main

  # Use personal SSH profile
  git remote add origin git@github-personal:MKAbdElrahman/issues.git

  git push -u origin main

  For Work Account

  echo "# issues" >> README.md
  git init
  git add README.md

  # Set user info for work account
  git config --local user.name "Your Work Name"
  git config --local user.email "work@company.com"

  git commit -m "first commit"
  git branch -M main

  # Use work SSH profile
  git remote add origin git@github-work:CompanyOrg/issues.git

  git push -u origin main

  Key Changes

  1. Replace: git@github.com: → git@github-personal: or git@github-work:
  2. Add: Git user config (git config --local user.name/email) before committing

  Quick Pattern

  The general pattern is:
  # Original
  git remote add origin git@github.com:username/repo.git

  # Personal
  git remote add origin git@github-personal:username/repo.git

  # Work
  git remote add origin git@github-work:username/repo.git

  This assumes you've already set up your ~/.ssh/config with the github-personal and github-work host
  aliases as shown in the documentation.
