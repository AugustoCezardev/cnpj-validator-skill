# CNPJ Alphanumeric Fix Guide

## Correct Validation Logic by Language

---

### Python

```python
import re

def validate_cnpj(cnpj: str) -> bool:
    """Validates alphanumeric CNPJ (with or without mask)."""
    # Remove mask
    cnpj = cnpj.strip().upper()
    cnpj = re.sub(r'[.\-/]', '', cnpj)
    
    if len(cnpj) != 14:
        return False
    
    # First 12 chars: alphanumeric; last 2: digits
    if not re.match(r'^[A-Z0-9]{12}[0-9]{2}$', cnpj):
        return False
    
    # Reject known invalid patterns (all same char)
    if len(set(cnpj)) == 1:
        return False
    
    def char_val(c):
        return int(c) if c.isdigit() else ord(c) - ord('A') + 10
    
    def calc_dv(base, weights):
        total = sum(char_val(c) * w for c, w in zip(base, weights))
        rem = total % 11
        return 0 if rem < 2 else 11 - rem
    
    weights1 = [5,4,3,2,9,8,7,6,5,4,3,2]
    weights2 = [6,5,4,3,2,9,8,7,6,5,4,3,2]
    
    dv1 = calc_dv(cnpj[:12], weights1)
    dv2 = calc_dv(cnpj[:12] + str(dv1), weights2)
    
    return cnpj[12:] == f'{dv1}{dv2}'

def format_cnpj(cnpj: str) -> str:
    """Apply mask to unmasked CNPJ."""
    c = re.sub(r'[.\-/]', '', cnpj.strip().upper())
    return f'{c[:2]}.{c[2:5]}.{c[5:8]}/{c[8:12]}-{c[12:14]}'

def strip_cnpj(cnpj: str) -> str:
    """Remove mask, return uppercase."""
    return re.sub(r'[.\-/]', '', cnpj.strip()).upper()
```

---

### JavaScript / TypeScript

```typescript
function charVal(c: string): number {
  return /\d/.test(c) ? parseInt(c, 10) : c.toUpperCase().charCodeAt(0) - 55;
}

function validateCNPJ(raw: string): boolean {
  const cnpj = raw.replace(/[.\-/]/g, '').toUpperCase();
  
  if (cnpj.length !== 14) return false;
  if (!/^[A-Z0-9]{12}\d{2}$/.test(cnpj)) return false;
  if (/^(.)\1+$/.test(cnpj)) return false; // all same char
  
  const calcDV = (base: string, weights: number[]): number => {
    const sum = base.split('').reduce((acc, c, i) => acc + charVal(c) * weights[i], 0);
    const rem = sum % 11;
    return rem < 2 ? 0 : 11 - rem;
  };
  
  const w1 = [5,4,3,2,9,8,7,6,5,4,3,2];
  const w2 = [6,5,4,3,2,9,8,7,6,5,4,3,2];
  
  const dv1 = calcDV(cnpj.slice(0, 12), w1);
  const dv2 = calcDV(cnpj.slice(0, 12) + dv1, w2);
  
  return cnpj.slice(12) === `${dv1}${dv2}`;
}

function stripCNPJ(cnpj: string): string {
  return cnpj.replace(/[.\-/]/g, '').toUpperCase();
}

function formatCNPJ(cnpj: string): string {
  const c = stripCNPJ(cnpj);
  return `${c.slice(0,2)}.${c.slice(2,5)}.${c.slice(5,8)}/${c.slice(8,12)}-${c.slice(12)}`;
}
```

---

### Java

```java
public class CnpjValidator {
    private static int charVal(char c) {
        return Character.isDigit(c) ? Character.getNumericValue(c) : (c - 'A' + 10);
    }

    private static int calcDV(String base, int[] weights) {
        int sum = 0;
        for (int i = 0; i < weights.length; i++) {
            sum += charVal(base.charAt(i)) * weights[i];
        }
        int rem = sum % 11;
        return rem < 2 ? 0 : 11 - rem;
    }

    public static boolean validate(String raw) {
        String cnpj = raw.replaceAll("[.\\-/]", "").toUpperCase();
        if (cnpj.length() != 14) return false;
        if (!cnpj.matches("[A-Z0-9]{12}[0-9]{2}")) return false;
        if (cnpj.chars().distinct().count() == 1) return false;

        int[] w1 = {5,4,3,2,9,8,7,6,5,4,3,2};
        int[] w2 = {6,5,4,3,2,9,8,7,6,5,4,3,2};

        int dv1 = calcDV(cnpj.substring(0, 12), w1);
        int dv2 = calcDV(cnpj.substring(0, 12) + dv1, w2);

        return cnpj.substring(12).equals(dv1 + "" + dv2);
    }
}
```

---

### PHP

```php
function validateCnpj(string $raw): bool {
    $cnpj = strtoupper(preg_replace('/[.\-\/]/', '', $raw));
    if (strlen($cnpj) !== 14) return false;
    if (!preg_match('/^[A-Z0-9]{12}[0-9]{2}$/', $cnpj)) return false;
    if (preg_match('/^(.)\1+$/', $cnpj)) return false;

    $charVal = fn($c) => ctype_digit($c) ? (int)$c : ord($c) - ord('A') + 10;
    
    $calcDV = function(string $base, array $weights) use ($charVal): int {
        $sum = 0;
        foreach (str_split($base) as $i => $c) {
            $sum += $charVal($c) * $weights[$i];
        }
        $rem = $sum % 11;
        return $rem < 2 ? 0 : 11 - $rem;
    };

    $w1 = [5,4,3,2,9,8,7,6,5,4,3,2];
    $w2 = [6,5,4,3,2,9,8,7,6,5,4,3,2];

    $dv1 = $calcDV(substr($cnpj, 0, 12), $w1);
    $dv2 = $calcDV(substr($cnpj, 0, 12) . $dv1, $w2);

    return substr($cnpj, 12) === "{$dv1}{$dv2}";
}
```

---

## Database Migration

### PostgreSQL

```sql
-- Change BIGINT/INT column to VARCHAR
ALTER TABLE companies ALTER COLUMN cnpj TYPE VARCHAR(18);
-- or if stored without mask:
ALTER TABLE companies ALTER COLUMN cnpj TYPE CHAR(14);

-- Add constraint for new format
ALTER TABLE companies ADD CONSTRAINT cnpj_format
  CHECK (cnpj ~ '^[A-Z0-9]{2}\.[A-Z0-9]{3}\.[A-Z0-9]{3}\/[A-Z0-9]{4}-[0-9]{2}$'
      OR cnpj ~ '^[A-Z0-9]{14}$');  -- unmasked
```

### MySQL

```sql
ALTER TABLE companies MODIFY cnpj VARCHAR(18) NOT NULL;
```

### SQL Server

```sql
ALTER TABLE companies ALTER COLUMN cnpj VARCHAR(18) NOT NULL;
```

---

## React / Input Mask

```tsx
// Using react-input-mask or similar
import InputMask from 'react-input-mask';

// formatChars must allow alphanumeric for A positions
<InputMask
  mask="aa.aaa.aaa/aaaa-99"
  formatChars={{ 'a': '[A-Za-z0-9]', '9': '[0-9]' }}
  value={cnpj}
  onChange={e => setCnpj(e.target.value.toUpperCase())}
/>
```

---

## Notes on `\D` in JavaScript

The regex `\D` (non-digit) is a **very common bug source**. It strips letters from CNPJ.

```javascript
// WRONG — strips 'A', 'B', etc.
cnpj.replace(/\D/g, '')

// CORRECT — only strip mask punctuation
cnpj.replace(/[.\-/]/g, '').toUpperCase()
```

Search your entire codebase for `/\D/g` usages near CNPJ handling.

---

## C# Complete Implementation

```csharp
using System;
using System.Text.RegularExpressions;

public static class CnpjValidator
{
    private static int CharVal(char c) =>
        char.IsDigit(c) ? (c - '0') : (char.ToUpper(c) - 'A' + 10);

    private static int CalcDV(string base12, int[] weights)
    {
        int sum = 0;
        for (int i = 0; i < weights.Length; i++)
            sum += CharVal(base12[i]) * weights[i];
        int rem = sum % 11;
        return rem < 2 ? 0 : 11 - rem;
    }

    public static bool Validate(string raw)
    {
        if (raw == null) return false;
        string cnpj = Regex.Replace(raw, @"[.\-/]", "").ToUpper();

        if (cnpj.Length != 14) return false;
        if (!Regex.IsMatch(cnpj, @"^[A-Z0-9]{12}\d{2}$")) return false;
        if (Regex.IsMatch(cnpj, @"^(.)\1+$")) return false; // all same char

        int[] w1 = { 5,4,3,2,9,8,7,6,5,4,3,2 };
        int[] w2 = { 6,5,4,3,2,9,8,7,6,5,4,3,2 };

        int dv1 = CalcDV(cnpj.Substring(0, 12), w1);
        int dv2 = CalcDV(cnpj.Substring(0, 12) + dv1, w2);

        return cnpj.Substring(12) == $"{dv1}{dv2}";
    }

    public static string Strip(string cnpj) =>
        Regex.Replace(cnpj?.Trim() ?? "", @"[.\-/]", "").ToUpper();

    public static string Format(string cnpj)
    {
        string c = Strip(cnpj);
        return $"{c[..2]}.{c[2..5]}.{c[5..8]}/{c[8..12]}-{c[12..14]}";
    }
}
```

### Entity Framework Migration

```csharp
// In your migration file
migrationBuilder.AlterColumn<string>(
    name: "Cnpj",
    table: "Companies",
    type: "varchar(18)",
    maxLength: 18,
    nullable: false,
    oldClrType: typeof(long),
    oldType: "bigint");
```

### Data Annotation for Models

```csharp
public class Company
{
    [Required]
    [MaxLength(18)]
    [RegularExpression(@"^[A-Z0-9]{2}\.[A-Z0-9]{3}\.[A-Z0-9]{3}\/[A-Z0-9]{4}-[0-9]{2}$",
        ErrorMessage = "CNPJ inválido")]
    public string Cnpj { get; set; }
}
```
