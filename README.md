---
name: cnpj-alpha-validator
description: >
  Validates codebases for compatibility with the new alphanumeric CNPJ format
  introduced by the Brazilian Receita Federal. Use this skill whenever the user
  asks to audit, scan, validate, or check code for CNPJ issues, CNPJ migration
  problems, alphanumeric CNPJ compatibility, or "novo CNPJ alfanumérico". Also
  trigger when the user mentions updating their ERP, NF-e emitter, or any
  Brazilian fiscal/tax system to support the new CNPJ format. This skill
  should be used proactively even if the user just says "check my code for
  CNPJ problems" or "will my system work with the new CNPJ?".
---

# CNPJ Alphanumeric Migration Validator

## Background

The Brazilian Receita Federal introduced alphanumeric CNPJs to avoid exhausting purely numeric combinations.

**Key facts:**
- CNPJ remains 14 characters with mask `XX.XXX.XXX/XXXX-DV`
- The **first 12 positions** can now contain uppercase letters (A–Z) AND digits (0–9)
- The **last 2 digits** (verification digits / DV) remain **numeric only**
- Existing numeric CNPJs are **unchanged** — no migration needed for existing companies
- All systems that read, validate, store, or emit CNPJs **must be updated**

**Example valid alphanumeric CNPJ:** `AB.CDE.FGH/0001-12`

---

## Workflow

### Step 1 — Understand the scope

Ask (or infer from context):
- What language(s) / frameworks are in the codebase?
- Is a file/directory provided, or should you search recursively?
- Are there NF-e / SEFAZ integrations?
- Is there a database schema to check?

### Step 1.5 — Discover CNPJ variable names

**Always ask the user** if they use custom variable/field names for CNPJ beyond the obvious ones. Many codebases store CNPJs under non-standard names such as:

> "Besides `cnpj`, does your codebase store CNPJ values under any other variable, column, or field names? For example: `document`, `doc`, `fiscal_id`, `tax_id`, `company_id`, `registro`, `inscricao`, `identificador`, `nf_emit`, `dest_cnpj`, `emit_cnpj`, `remetente`, `destinatario`, `fornecedor_doc`, etc."

Collect all confirmed names and **add them to every grep command** in Step 2. For example, if the user says they also use `tax_id` and `fiscal_doc`:

```bash
grep -rn -E "(cnpj|tax_id|fiscal_doc)" <dir> | head -300
```

Also do a broad scan for any field that stores 14-character strings to catch undiscovered names:
```bash
# Catches VARCHAR(14), CHAR(14), maxlength=14 near fiscal-sounding names
grep -rn -iE "(varchar|char|maxlength|max_length)\s*[\(\=]\s*1[48]" <dir>
```

### Step 1.6 — Discover DB mapping aliases and column links

Always scan ORM/data-mapper files and SQL aliases for CPF/CNPJ-related names, even when property names are different.

Mandatory patterns to search for:
- `NR_CPFCNPJ`
- `CPF`
- `CNPJ`
- `CPFCNPJ`
- `NR_CHAVE_UNICA`
- any alias/column containing `DOC`, `DOCUMENT`, `FISCAL`, `TAX` when near customer/person fields

Why this is required:
- many breakages happen in mapping layers where a numeric property is linked to an alphanumeric document column.
- examples: `Map(i => i.IdCliente).ToColumn("NR_CPFCNPJ")`, SQL aliases like `... NR_CPFCNPJ IdCliente`.

Quick mapping grep examples:
```bash
grep -rn -iE "NR_CPFCNPJ|CPF|CNPJ|CPFCNPJ|NR_CHAVE_UNICA|ToColumn\(|Map\(" <dir>
grep -rn -iE "\bAS\b\s+.*(CPF|CNPJ|CPFCNPJ)|\b(CPF|CNPJ|CPFCNPJ)\b\s+[A-Za-z_][A-Za-z0-9_]*" <dir>
```

### Step 2 — Scan the codebase

Run targeted searches for the patterns listed in `references/patterns.md`, using ALL variable names gathered in Step 1.5 and all mapping aliases/column links found in Step 1.6.

```bash
# Quick broad scan — adapt as needed (replace CNPJ_NAMES with all discovered names)
CNPJ_NAMES="cnpj|tax_id|fiscal_id"  # extend with user-provided names
grep -rn --include="*.py" --include="*.js" --include="*.ts" \
         --include="*.java" --include="*.php" --include="*.rb" \
         --include="*.go" --include="*.cs" --include="*.sql" \
         -E "($CNPJ_NAMES)" <target_dir> | head -200
```

Load `references/patterns.md` for the full list of anti-patterns to grep for.

### Step 3 — Classify findings

For each finding, classify as:

| Severity | Meaning |
|----------|---------|
| 🔴 Critical | Will break or reject valid alphanumeric CNPJs |
| 🟡 Warning | May break in edge cases or under new format |
| 🔵 Info | Cosmetic / should be reviewed but not necessarily wrong |

See `references/patterns.md` for severity mapping of each pattern.

### Step 4 — Report

Produce a structured report (see Output Format below).

### Step 5 — Suggest fixes

For each critical/warning finding, provide a corrected code snippet using the rules in `references/fix-guide.md`.

---
## Output Format

```
## CNPJ Alphanumeric Compatibility Report

### Summary
- Files scanned: N
- Critical issues: N
- Warnings: N
- Info: N

### Critical Issues 🔴
#### [FILE:LINE] Short description
**Problem:** ...
**Current code:** `...`
**Fix:** `...`

### Warnings 🟡
...

### Info 🔵
...

### Recommendations
...
```

---

## Reference Files

- `references/patterns.md` — Anti-patterns by language with severity
- `references/fix-guide.md` — Correct validation logic, regex, and DB types

Read these before scanning if the user's codebase uses a specific language you need patterns for.
