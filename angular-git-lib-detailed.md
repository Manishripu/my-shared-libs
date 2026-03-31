# 📦 Angular Library Sharing via Git (Repo A → Repo B)

## 🎯 Objective
Create a reusable Angular 19 library in **Repo A** and consume it in **Repo B** using **Git-based installation (no registry required)**.

---

# 🧭 High-Level Flow

Repo A (Library)
   ↓ build (dist)
   ↓ commit + tag
Git Repository
   ↓ install via git tag
Repo B (Consumer App)

---

# 🧱 PHASE 1: Create Angular Library (Repo A)

## 1. Generate Library
ng generate library common-lib

## 2. Structure
projects/common-lib/src/lib/
  components/
  services/
  models/
  utils/

## 3. Export Public API
projects/common-lib/src/public-api.ts

export * from './lib/common-lib.module';
export * from './lib/services/api.service';

---

# 🏗️ PHASE 2: Build Library

ng build common-lib

Output:
dist/common-lib/

---

# 📦 PHASE 3: Make Repo Installable

## Allow dist
.gitignore → !dist/common-lib

## Commit build
git add dist/common-lib
git commit -m "build: add compiled lib"

## Root package.json
{
  "name": "@your-org/common-lib",
  "version": "1.0.0",
  "main": "dist/common-lib/bundles/common-lib.umd.js",
  "module": "dist/common-lib/fesm2022/common-lib.mjs",
  "types": "dist/common-lib/index.d.ts"
}

---

# 🏷️ PHASE 4: Versioning

git tag v1.0.0
git push origin v1.0.0

Update:
ng build common-lib
git add dist/common-lib
git commit -m "update"
git tag v1.1.0
git push origin --tags

---

# 📥 PHASE 5: Use in Repo B

npm install git+https://github.com/your-org/repo-a.git#v1.0.0

package.json:
"@your-org/common-lib": "git+https://github.com/your-org/repo-a.git#v1.0.0"

Import:
import { CommonLibModule } from '@your-org/common-lib';

---

# 🔄 PHASE 6: Update

npm install git+https://github.com/your-org/repo-a.git#v1.1.0

---

# 🧠 BEST PRACTICES

- Always use git tags
- Commit dist
- Rebuild before tagging
- Use peerDependencies
- Keep lib decoupled

---

# ❌ COMMON MISTAKES

- Not committing dist
- Using main branch
- Missing exports
- Tight coupling

---

# ⚠️ LIMITATIONS

- No strict versioning
- Slower installs
- Larger repo
- Dependency conflicts

---

# 🏁 SUMMARY

✔ No registry needed  
✔ Easy setup  
⚠️ Requires discipline  
