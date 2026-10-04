
## Executive Summary

This case study documents practical security and code testing of the **public, non-authenticated surface of a real Norwegian website**. The organization name and identifying production details have intentionally been removed.

The assessment identified several reproducible defects and security-relevant behaviors:

- Incorrect handling of reserved characters and URL-encoded values
- HTTP Parameter Pollution (HPP) behavior
- HTTP 500 responses caused by malformed parameters
- Inconsistent API input validation
- Content Security Policy (CSP) observations

These issues can cause requests to differ from user input, alter query structure, create inconsistent behavior between application layers, or cause frontend code expecting JSON to receive an HTML server-error response.

Testing was performed using public functionality, HTTP requests/responses, public APIs and publicly delivered JavaScript, following **OWASP WSTG** concepts.

No authentication bypass, private-data access, destructive actions, brute force or denial-of-service testing was performed.

**No Critical/High vulnerability, SQL injection, XSS or authorization bypass was confirmed.**

---

## Finding 1 — Search Parameter Encoding

**Classification:** Confirmed defect / potential security-relevant behavior  
**Severity:** Low  
**Related:** OWASP WSTG-INPV-04, CWE-116 / CWE-20

A public search function did not consistently preserve reserved characters supplied as user input.

### Test

    Input:
    TEST_A&TEST_B

    Expected:
    The complete string remains one search value.

    Observed:
    "&" could be interpreted as URL syntax,
    changing the effective request.

Similar behavior was observed with `#`.

Public JavaScript analysis identified a URL-building path where parameters were concatenated without consistently using safe parameter serialization.

**Potential impact:** altered search requests, unintended query parameters and inconsistent frontend/backend state.

No authorization bypass or private-data access was demonstrated.

---

## Finding 2 — Encoded Values Modified by API

**Classification:** Confirmed input-processing defect  
**Severity:** Low

Correctly URL-encoded reserved characters were not always preserved during processing.

### Tested values

    %23 → #
    %26 → &
    %2B → +

Observed behavior included truncation or transformation of the effective value.

This suggests that the issue may involve more than frontend URL construction and could occur during decoding or parameter processing between application layers.

The exact backend cause was not established.

---

## Finding 3 — Invalid Parameters Trigger HTTP 500

**Classification:** Confirmed validation / error-handling defect  
**Severity:** Informational / Low

### Control

    page=0
    → HTTP 200
    → JSON response

### Invalid input

    page=-1
    → HTTP 500

    page=invalid
    → HTTP 500

Invalid filter values also produced HTTP 500 responses.

**Potential impact:**

- Frontend expects JSON but receives HTML
- Response parsing can fail
- Invalid client input reaches server-error handling
- API behavior becomes inconsistent

A controlled `400 Bad Request` response would normally be preferable.

No denial-of-service impact was tested or demonstrated.

---

## Finding 4 — HTTP Parameter Pollution Candidate

**Classification:** Potential vulnerability  
**Severity:** Low

Duplicate query parameters were handled inconsistently.

    parameter=value1&parameter=value2

Some duplicated parameters produced HTTP 500 responses, while another duplicated parameter was combined into a compound value.

Different interpretations of duplicate parameters between application layers can create security-relevant behavior.

No authentication bypass, authorization bypass or protected-data access was demonstrated.

---

## Finding 5 — Inconsistent API Error Handling

Different public APIs demonstrated different validation behavior.

Some endpoints correctly returned:

- HTTP 400 for invalid requests
- HTTP 404 for unknown resources

Another tested API returned **HTTP 500** for comparable malformed parameters.

This helped distinguish a specific validation/robustness defect from normal platform behavior.

---

## Finding 6 — CORS Review

Controlled foreign `Origin` values were tested against selected public endpoints.

The tested responses did not expose permissive `Access-Control-Allow-Origin` or credential-sharing behavior.

**Result:** No permissive CORS vulnerability was demonstrated.

---

## Finding 7 — Input Reflection / XSS Checks

Neutral special characters were tested:

    < > " '

The tested contexts escaped these characters instead of rendering them as executable markup.

**Result:** No unescaped reflection was demonstrated in the tested contexts.

Executable XSS payloads were not used, so this does not establish that every possible XSS path is safe.

---

## Finding 8 — CSP / Security Configuration

Content Security Policy and related HTTP security configuration were reviewed.

Defense-in-depth observations included directives such as:

    'unsafe-inline'
    'unsafe-eval'

and inconsistent CSP coverage across selected public pages.

These are **security configuration observations**, not proof of an exploitable XSS vulnerability.

---

## Additional Security Checks

### SQL Injection

Neutral quotation-mark input did not produce SQL errors, SQLSTATE output or database stack traces.

**Result:** SQL injection was not confirmed.

### NoSQL Injection

A JSON-like marker remained normal string data in the observed response.

**Result:** NoSQL injection was confirmed.

### Server-Side Template Injection

Neutral template-style markers did not demonstrate server-side evaluation.

**Result:** SSTI was not confirmed.

---

## Overall Result

The assessment identified reproducible defects involving:

- URL and parameter handling
- Encoded input processing
- HTTP 500 error handling
- Duplicate parameters / HPP behavior
- API input validation
- Defensive security configuration

The assessment did **not** establish a Critical/High exploit, authentication bypass, private-data exposure, SQL injection or XSS.

This case study demonstrates practical **Web/API Security Testing, public JavaScript analysis, reproducible testing, OWASP methodology, technical documentation, and responsible classification of security findings**.