# ALU Regex Data Extraction & Secure Validation

A Python program that extracts and validates eight types of structured data
from raw, production-style text using regular expressions. All input is
treated as untrusted, and security considerations are applied at every step.

---

## Project Structure

```
alu-regex-data-extraction/
├── input/
│   └── raw-text.txt        ← Realistic messy input (HR records, payments, HTML, etc.)
├── src/
│   └── main.py             ← Extraction engine with all regex patterns + security logic
├── output/
│   └── sample-output.json  ← Structured JSON output from a full run
└── README.md
```

---

## Requirements

- Python 3.10 or higher (uses the `X | Y` union type hint)
- No third-party libraries — only the Python standard library (`re`, `json`,
  `unicodedata`, `pathlib`, `datetime`)

---

## How to Run

```bash
# From the project root directory:
python3 src/main.py
```

The program will:
1. Read `input/raw-text.txt`
2. Print a summary report to the console
3. Write full structured results to `output/sample-output.json`

---

## Data Types Extracted

| # | Type | Details |
|---|------|---------|
| 1 | **Email addresses** | General + ALU-specific domain validation |
| 2 | **URLs** | `http`/`https` only; path injection blocked |
| 3 | **Phone numbers** | E.164 (7–15 digits); international and local formats |
| 4 | **Credit card numbers** | Visa, MC, Amex, Discover; Luhn-checked; masked in output |
| 5 | **Time (12h & 24h)** | `09:00 AM`, `14:00` etc.; invalid times rejected |
| 6 | **HTML tags** | Classified as safe / dangerous / suspicious |
| 7 | **Hashtags** | Must start with a letter; pure-numeric tags rejected |
| 8 | **Currency amounts** | `$`, `€`, `£`; leading or trailing symbol |

---

## ALU-Specific Email Validation

Three domain patterns are validated independently:

| Category | Domain | Example |
|----------|--------|---------|
| Staff    | `@alueducation.com` | `amara.diallo@alueducation.com` |
| Alumni   | `@alumni.alueducation.com` | `tunde.oke@alumni.alueducation.com` |
| SI       | `@si.alueducation.com` | `lena.fischer@si.alueducation.com` |

All three patterns additionally reject:
- Addresses with no local part (e.g. `@alueducation.com`)
- Consecutive dots in local part or domain (e.g. `james..bond@...`)
- Double `@` signs
- Any value containing an injection marker

---

## Security Design

### 1. Input Sanitisation
Every line is passed through `sanitise_line()` before pattern matching.
This function strips all Unicode control characters (null bytes, carriage
returns, ANSI escapes) that are used in HTTP response-splitting and null-byte
injection attacks.

### 2. Injection Deny-List
`is_safe(value)` applies a deny-list regex to any extracted value before
accepting it. Rejected markers include:
- XSS vectors: `<script`, `onerror=`, `javascript:`, `<iframe`
- SQL injection: `DROP TABLE`, `UNION SELECT`, `'--`, `;--`
- Code execution: `eval(`, `exec(`, `fetch(`
- Data exfiltration probes: `document.cookie`, `window.location`

### 3. Context-Aware URL Filtering
URLs are not extracted from lines that contain injection markers. This
catches URLs embedded inside `<script>fetch(...)`, `<iframe onerror=...>`,
and similar hostile contexts — even when the URL substring itself looks
benign.

### 4. Credit Card Masking (PCI-DSS 3.3)
Credit card numbers are **never stored or displayed in plain text**.
The `mask_card()` function replaces all but the last four digits with `*`
before the value appears in any output or log:

```
4111 1111 1111 1111  →  **** **** **** 1111
```

### 5. Luhn Algorithm Validation
All matched card candidates are run through the ISO/IEC 7812 Luhn checksum
before being accepted. Cards that fail (e.g. `9999 9999 9999 9999`) are
moved to the `rejected` list with the reason `"failed Luhn check"`.

### 6. Phone Number Validation
After regex matching, digit count is verified to be within the E.164 range
(7–15 digits). A secondary filter rejects strings that match the pattern
`NNNN NNNN NNNN` (three groups of four digits), which are partial card
numbers rather than phone numbers.

### 7. HTML Tag Classification
Every matched HTML tag is classified:
- **safe** — standard display tags with no dangerous attributes
- **dangerous** — `<script>`, `<iframe>`, `<object>`, `<embed>`, etc.
- **suspicious** — any tag with an `on*=` event handler or `javascript:` href

### 8. No Code Evaluation
The program never calls `eval()`, `exec()`, `subprocess`, or any dynamic
code runner on input content. All input is treated as data, not code.

---

## Sample Output (excerpt)

```json
{
  "emails": {
    "alu_staff": [
      "records@alueducation.com",
      "amara.diallo@alueducation.com"
    ],
    "alu_alumni": ["tunde.oke@alumni.alueducation.com"],
    "alu_si":    ["lena.fischer@si.alueducation.com"],
    "other":     ["info@partner-university.edu"],
    "rejected":  [{"value": "james..bond@alueducation.com", "reason": "consecutive dots"}]
  },
  "credit_cards": {
    "valid":    [{"masked": "**** **** **** 1111", "luhn_pass": true}],
    "rejected": [{"masked": "**** **** **** 9999", "reason": "failed Luhn check"}]
  },
  "html_tags": {
    "safe":      ["<h1>", "<p>", "<strong>"],
    "dangerous": ["<script>", "<iframe src=\"https://phishing.example.com\">"],
    "suspicious": ["<img src=\"x\" onerror=\"...\">"]
  }
}
```

---

## What the Input File Contains

`input/raw-text.txt` is modelled on a real-world internal communications
digest. It intentionally includes:

- Valid and invalid ALU staff, alumni, and SI email addresses
- Payment transaction records with real and fake credit card numbers
- Curated resource URLs alongside malformed and injection-containing URLs
- Event schedules with 12-hour and 24-hour times (including one invalid time)
- Phone numbers in international, North-American, and local African formats
- Raw HTML fragments from a CMS, including a set of dangerous/suspicious tags
- Social media hashtags with edge cases (empty, numeric-only)
- Currency amounts in USD, EUR, and GBP
- Deliberate injection payloads: XSS, SQL injection, HTTP response-splitting,
  and card-field comment injection — to verify the security filters work
