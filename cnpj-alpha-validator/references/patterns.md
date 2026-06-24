# CNPJ Anti-Patterns Reference

## What is a valid alphanumeric CNPJ?

```
Format:  XX.XXX.XXX/XXXX-DV
Chars:   [A-Z0-9]{2}.[A-Z0-9]{3}.[A-Z0-9]{3}/[A-Z0-9]{4}-[0-9]{2}
Unmasked:[A-Z0-9]{12}[0-9]{2}   (14 chars total)
```

Uppercase letters only. DVs (last 2) are always digits.

---

## Pattern Categories

### 1. Regex Validation — 🔴 Critical

Patterns that only accept digits in the first 12 positions.

```regex
# WRONG — only digits
^\d{2}\.\d{3}\.\d{3}\/\d{4}-\d{2}$
^\d{14}$
[0-9]{14}
\d{2}\.\d{3}\.\d{3}\/\d{4}\-\d{2}

# CORRECT — alphanumeric first 12, digits for DV
^[A-Z0-9]{2}\.[A-Z0-9]{3}\.[A-Z0-9]{3}\/[A-Z0-9]{4}-[0-9]{2}$
^[A-Z0-9]{12}[0-9]{2}$   # unmasked
```

Grep for these in source:
```bash
grep -rn --include="*.{js,ts,py,java,php,rb,go,cs}" \
  -E '\\d\{14\}|\\d\{2\}\\\.\\d\{3\}' <dir>
```

---

### 2. Numeric Type Casting — 🔴 Critical

Storing or parsing CNPJ as an integer/number.

**Python:**
```python
# WRONG
cnpj = int(cnpj_str)
cnpj = int(cnpj.replace('.','').replace('/','').replace('-',''))

# CORRECT — keep as string
cnpj = cnpj_str.strip()
```

**JavaScript/TypeScript:**
```javascript
// WRONG
const cnpj = Number(cnpjStr);
const cnpj = parseInt(cnpjStr.replace(/\D/g, ''), 10);
// Note: parseInt is OK for DV-only extraction, NOT for the full CNPJ

// CORRECT
const cnpj = cnpjStr.trim();
```

**Java:**
```java
// WRONG
long cnpj = Long.parseLong(cnpjStr);

// CORRECT
String cnpj = cnpjStr.trim();
```

Grep:
```bash
grep -rn -E "(int|long|Integer|Long|parseInt|parseLong|Number|int64|int32|Convert\.ToInt|long\.Parse|int\.Parse)\s*\(?\s*cnpj" <dir>
```

**C#:**
```csharp
// WRONG
long cnpj = long.Parse(cnpjStr);
int cnpj = Convert.ToInt32(cnpjStr);
var cnpj = Convert.ToInt64(cnpjStr.Replace(".","").Replace("/","").Replace("-",""));

// CORRECT
string cnpj = cnpjStr.Trim();
```

Grep for C#:
```bash
grep -rn -iE "(long\.Parse|int\.Parse|Convert\.ToInt|Convert\.ToInt64)\s*\(?.*cnpj" <dir>
```

---

### 3. Database Column Types — 🔴 Critical

```sql
-- WRONG
cnpj BIGINT
cnpj INT
cnpj NUMERIC(14)
cnpj DECIMAL(14)

-- CORRECT
cnpj VARCHAR(18)   -- with mask
cnpj CHAR(14)      -- without mask (14 alphanumeric chars)
cnpj VARCHAR(14)   -- without mask
```

Also check ORM model fields:

**Django:**
```python
# WRONG
cnpj = models.IntegerField()
cnpj = models.BigIntegerField()

# CORRECT
cnpj = models.CharField(max_length=18)
```

**Hibernate/JPA:**
```java
// WRONG
@Column(columnDefinition = "BIGINT")
private Long cnpj;

// CORRECT
@Column(length = 18)
private String cnpj;
```

**Entity Framework (C#):**
```csharp
// WRONG
[Column(TypeName = "bigint")]
public long Cnpj { get; set; }

// CORRECT
[Column(TypeName = "varchar(18)")]
[MaxLength(18)]
public string Cnpj { get; set; }
```

Grep:
```bash
grep -rn -iE "cnpj.*(bigint|int\b|numeric|decimal|long|integer)" <dir>
grep -rn -iE "(bigint|int\b|numeric|decimal).*(cnpj)" <dir>
grep -rn -iE "TypeName.*bigint|public\s+long\s+[Cc]npj" <dir>   # C#
```

---

### 4. String Sanitization Removing Letters — 🔴 Critical

```python
# WRONG — strips letters from CNPJ
cnpj = re.sub(r'[^0-9]', '', cnpj)
cnpj = ''.join(filter(str.isdigit, cnpj))
cnpj = cnpj.replace(...).isdigit()

# CORRECT — keep alphanumeric, only remove mask characters
cnpj_clean = re.sub(r'[.\-/]', '', cnpj).upper()
# or keep only valid chars
cnpj_clean = re.sub(r'[^A-Z0-9]', '', cnpj.upper())
```

```javascript
// WRONG
cnpj = cnpj.replace(/\D/g, '');   // \D removes ALL non-digits, including letters!

// CORRECT
cnpj = cnpj.replace(/[.\-\/]/g, '').toUpperCase();
// or
cnpj = cnpj.replace(/[^A-Z0-9]/gi, '').toUpperCase();
```

```csharp
// WRONG — strips letters
cnpj = Regex.Replace(cnpj, @"[^\d]", "");
cnpj = Regex.Replace(cnpj, @"\D", "");
cnpj = new string(cnpj.Where(char.IsDigit).ToArray());

// CORRECT — remove only mask punctuation
cnpj = Regex.Replace(cnpj, @"[.\-/]", "").ToUpper();
// or keep only valid chars
cnpj = Regex.Replace(cnpj, @"[^A-Z0-9]", "", RegexOptions.IgnoreCase).ToUpper();
```

Grep:
```bash
grep -rn -E "replace.*\\\\D|\\[\\^0-9\\]|isdigit|filter.*digit|IsDigit|\\\\D" <dir>
grep -rn -iE "Where\(char\.IsDigit\)|Where\(c\s*=>" <dir>   # C# LINQ digit filter
```

---

### 5. Verification Digit (DV) Calculation — 🔴 Critical

The DV algorithm must use ASCII/character code values for letters, not just numeric values. The weights and modulus remain the same, but each character's numeric value is `ord(char) - 48` for digits and `ord(char) - 55` for uppercase letters (i.e., A=10, B=11, ..., Z=35).

```python
# WRONG — only works for numeric CNPJ
def calc_dv(cnpj):
    nums = [int(c) for c in cnpj]   # int('A') raises ValueError
    ...

# CORRECT
def char_value(c):
    """Returns numeric value of a CNPJ character (0-9 → 0-9, A-Z → 10-35)"""
    if c.isdigit():
        return int(c)
    return ord(c.upper()) - ord('A') + 10

def calc_dv(cnpj_12):
    """Calc both DVs for a 12-char alphanumeric CNPJ base."""
    weights1 = [5,4,3,2,9,8,7,6,5,4,3,2]
    weights2 = [6,5,4,3,2,9,8,7,6,5,4,3,2]
    
    total = sum(char_value(c) * w for c, w in zip(cnpj_12, weights1))
    dv1 = 0 if (total % 11) < 2 else 11 - (total % 11)
    
    total = sum(char_value(c) * w for c, w in zip(cnpj_12, weights2)) + dv1 * 2
    dv2 = 0 if (total % 11) < 2 else 11 - (total % 11)
    
    return str(dv1) + str(dv2)
```

Grep for DV calculation functions that use `int(c)` directly:
```bash
grep -rn -iE "def.*cnpj|function.*cnpj|cnpj.*dv|valida.*cnpj|validate.*cnpj" <dir>
```

---

### 6. Input Field Masking — 🟡 Warning

Frontend input masks that only accept digits.

```javascript
// WRONG — jQuery Mask Plugin
$('#cnpj').mask('99.999.999/9999-99');  // '9' = digit only

// CORRECT
$('#cnpj').mask('AA.AAA.AAA/AAAA-99', {  // 'A' = alphanumeric, '9' = digit
    translation: {
        'A': { pattern: /[A-Za-z0-9]/ }
    }
});
```

```html
<!-- WRONG -->
<input type="number" name="cnpj" />
<input pattern="\d{14}" />

<!-- CORRECT -->
<input type="text" name="cnpj" maxlength="18" pattern="[A-Z0-9]{2}\.[A-Z0-9]{3}\.[A-Z0-9]{3}\/[A-Z0-9]{4}-[0-9]{2}" />
```

Grep:
```bash
grep -rn -iE "mask.*cnpj|cnpj.*mask|9{2}\.9{3}|99\.999\.999" <dir>
grep -rn -iE 'type="number".*cnpj|cnpj.*type="number"' <dir>
```

---

### 7. `.length` / `len()` checks — 🟡 Warning

These are fine IF the CNPJ is kept as string and mask characters handled correctly.

```python
# OK only if cnpj is always stored without mask
if len(cnpj) != 14:
    raise ValueError("Invalid CNPJ length")

# Risky — could break if mask sometimes included
if len(cnpj.replace('.','').replace('/','').replace('-','')) != 14:
    ...
```

Flag any length check that also does digit-only stripping.

---

### 8. Hardcoded "all zeros" / test CNPJs — 🔵 Info

```python
# These are common test patterns — review if they need alphanumeric equivalents
INVALID_CNPJS = ['00000000000000', '11111111111111', ...]
```

---

### 9. Third-party libraries — 🟡 Warning

Check if any CNPJ validation library is used and whether it has been updated.

Common libs to audit:
- **Python:** `validate-docbr`, `brutils`, `python-cnpj`
- **JavaScript/TypeScript:** `cpf-cnpj-validator`, `@fnando/cnpj`, `validar-cnpj`
- **PHP:** `respect/validation`
- **Ruby:** `cpf_cnpj`
- **Java:** `caelum-stella`
- **C#/.NET:** `BrazilianDocuments`, `Br.Cnpj`, `CpfCnpjUtils`, any NuGet package with "cpf" or "cnpj" in the name

Grep:
```bash
grep -rn -iE "cpf.cnpj|validate.docbr|brutils|caelum.stella|respect.validation|BrazilianDoc|Br\.Cnpj" <dir>
grep -rn -iE "cnpj|cpf" *.csproj **/*.csproj 2>/dev/null   # C# NuGet deps
```

---

### 10. C#-Specific Patterns — 🔴 Critical / 🟡 Warning

#### Regex with `\d` shorthand — 🔴 Critical

```csharp
// WRONG
var regex = new Regex(@"^\d{14}$");
var regex = new Regex(@"^\d{2}\.\d{3}\.\d{3}\/\d{4}-\d{2}$");
if (Regex.IsMatch(cnpj, @"\d{14}")) { ... }

// CORRECT
var regex = new Regex(@"^[A-Z0-9]{12}\d{2}$");
var regex = new Regex(@"^[A-Z0-9]{2}\.[A-Z0-9]{3}\.[A-Z0-9]{3}\/[A-Z0-9]{4}-\d{2}$");
```

Grep:
```bash
grep -rn -E '\\\\d\{14\}|\\\\d\{2\}' <dir> --include="*.cs"
```

#### `long` / `int` property types — 🔴 Critical

```csharp
// WRONG — in models, DTOs, entities
public long Cnpj { get; set; }
public int Cnpj { get; set; }
public ulong Cnpj { get; set; }

// CORRECT
public string Cnpj { get; set; }
```

Grep:
```bash
grep -rn -iE "public\s+(long|int|ulong|uint|Int64|Int32)\s+[Cc]npj" <dir> --include="*.cs"
```

#### JSON deserialization to numeric type — 🔴 Critical

```csharp
// WRONG — if the incoming JSON has an alphanumeric CNPJ, this will throw
public class CompanyDto {
    [JsonProperty("cnpj")]
    public long Cnpj { get; set; }   // breaks on "AB1234567890-01"
}

// CORRECT
public class CompanyDto {
    [JsonProperty("cnpj")]
    public string Cnpj { get; set; }
}
```

Grep:
```bash
grep -rn -iE "\[JsonProperty.*cnpj|\"cnpj\"\]\s*public\s+(long|int)" <dir> --include="*.cs"
```

#### `string.Format` / `ToString` with numeric format — 🟡 Warning

```csharp
// WRONG — only valid for all-digit CNPJs
cnpj.PadLeft(14, '0');
cnpj.ToString("D14");

// CORRECT — just ensure uppercase and correct length
cnpj = cnpj.Trim().ToUpper();
if (cnpj.Length != 14) throw new ArgumentException("CNPJ deve ter 14 caracteres");
```

Grep:
```bash
grep -rn -iE 'PadLeft.*cnpj|cnpj.*PadLeft|ToString\("D14"\)' <dir> --include="*.cs"
```

#### `char.IsDigit` / `char.IsNumber` LINQ filters — 🔴 Critical

```csharp
// WRONG — strips letters
var digits = cnpj.Where(char.IsDigit).ToArray();
var clean = new string(cnpj.Where(c => char.IsDigit(c)).ToArray());

// CORRECT
var clean = Regex.Replace(cnpj, @"[.\-/]", "").ToUpper();
```

Grep:
```bash
grep -rn -iE "Where\(char\.IsDigit|Where\(c\s*=>\s*char\.IsDigit" <dir> --include="*.cs"
```
