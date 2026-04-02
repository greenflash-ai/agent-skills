---
name: test-preprocess
description: Test shell preprocessing
user-invocable: true
---

# Preprocessing Test

Env test: !`printenv GREENFLASH_API_KEY 2>/dev/null || echo "NOT_SET"`
File test: !`head -1 .greenflash 2>/dev/null || echo "NO_FILE"`
Skill dir test: ${CLAUDE_SKILL_DIR}
