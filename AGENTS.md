# 🚀 AGENTS.MD - CEO EXECUTIVE MANDATE (V2026.02)

**Version:** 2026.02 "UNIVERSAL STANDARD"  
**Status:** ACTIVE - MUST READ BEFORE EVERY SESSION  
**Scope:** ALL PROJECTS

---

## 🔑 TOP 10 EXECUTIVE RULES

### 1. **PARALLEL EXECUTION MANDATE**
- ❌ `run_in_background=false` → NEVER
- ✅ `run_in_background=true` → ALWAYS

### 2. **SEARCH BEFORE CREATE**
- ❌ Blind file creation → NEVER
- ✅ `glob()`, `grep()` first → ALWAYS

### 3. **VERIFY-THEN-EXECUTE**
- ❌ Trust without verification → NEVER
- ✅ `lsp_diagnostics`, `bash` check → ALWAYS

### 4. **GIT COMMIT DISCIPLINE**
- ✅ After every significant change
- ✅ Conventional Commits (`feat:`, `fix:`, `docs:`)

### 5. **FREE-FIRST PHILOSOPHY**
- ✅ Self-hosted, free tiers, open source first

### 6. **RESOURCE PRESERVATION**
- ❌ Delete config files → NEVER

### 7. **NO-SCRIPT MANDATE**
- ❌ Manual bash scripts → NEVER  
- ✅ Use AI agents for everything → ALWAYS

### 8. **TODO DISCIPLINE**
- ✅ Create todos for multi-step tasks (`todowrite`)

### 9. **DOCUMENTATION FIRST**
- ✅ Update `BLUEPRINT.md` and `lastchanges.md` constantly

### 10. **PERFORMANCE FIRST**
- ✅ Native CDP over Playwright
- ✅ Optimized Docker containers

---

## 🚨 CRITICAL MANDATES

### DEQLHI-LOOP (INFINITE WORK MODE)
- After each completed task → Add 5 new tasks immediately
- Never "done" - only "next task"
- Always document → Every change in files

### PORT SOVEREIGNTY (NO STANDARD PORTS)
- Standard ports cause conflicts (3000, 5432, 8080, etc.)
- Use unique ports in 50000-59999 range
- Container naming: `{CATEGORY}-{NUMBER}-{NAME}`

---

## 📝 CODING STANDARDS

### TypeScript Configuration
```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "alwaysStrict": true,
    "strictNullChecks": true
  }
}
```

### Error Handling
```typescript
// CORRECT
try {
  const result = await riskyOperation();
  return result;
} catch (error) {
  logger.error('Operation failed', { error, context });
  throw new CustomError('Descriptive message', { cause: error });
}
```

---

## 🔄 WORKFLOW PROTOCOL

1. **Read Context:** `BLUEPRINT.md`, `lastchanges.md`
2. **Plan:** Create Todos
3. **Execute:** Parallel Agents
4. **Verify:** Tests & Diagnostics
5. **Document:** Update Docs
6. **Commit:** Git Push

---
*Authorized by Sisyphus - CEO of Code*
