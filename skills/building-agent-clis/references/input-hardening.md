# Input Hardening Guide

Defensive validation patterns for CLIs that agents invoke.

## Contents
- Why Agents Need Different Validation
- Validation Functions
- Testing Agent-Specific Failures

## Why Agents Need Different Validation

Humans typo. Agents hallucinate. The failure modes are completely different:

- A human types `../../.ssh` by accident — virtually never happens
- An agent generates `../../.ssh` by confusing path segments — plausible
- An agent embeds `?fields=name` inside a resource ID — has happened
- An agent pre-URL-encodes a string that gets double-encoded — common

Design validation for the mistakes agents actually make, not just the mistakes humans make.

## Validation Functions

### Path Safety

```python
import os

def validate_safe_path(path: str, allowed_root: str = None) -> str:
    """Canonicalize path and ensure it doesn't escape the sandbox."""
    if allowed_root is None:
        allowed_root = os.getcwd()

    # Resolve to absolute, following symlinks
    resolved = os.path.realpath(os.path.join(allowed_root, path))

    # Must be under allowed root
    if not resolved.startswith(os.path.realpath(allowed_root)):
        raise ValueError(
            f"Path '{path}' resolves outside allowed directory. "
            f"All paths must be within {allowed_root}"
        )

    return resolved
```

### Control Character Rejection

```python
def reject_control_chars(value: str, field_name: str) -> str:
    """Reject invisible/control characters that agents sometimes generate."""
    for i, char in enumerate(value):
        if ord(char) < 0x20 and char not in ('\n', '\r', '\t'):
            raise ValueError(
                f"'{field_name}' contains control character at position {i} "
                f"(0x{ord(char):02x}). Remove non-printable characters."
            )
    return value
```

### Resource ID Validation

```python
import re

def validate_resource_id(value: str, field_name: str) -> str:
    """Reject IDs with embedded query params or encoding artifacts."""
    # Agents commonly embed query params in IDs
    if '?' in value or '#' in value:
        raise ValueError(
            f"'{field_name}' contains '?' or '#'. "
            f"Resource IDs must not include query parameters. "
            f"Got: '{value}'"
        )

    # Agents commonly pre-encode strings
    if '%' in value:
        raise ValueError(
            f"'{field_name}' contains '%' (possible pre-encoding). "
            f"Pass the raw ID without URL encoding. Got: '{value}'"
        )

    return value
```

### JSON Payload Validation

```python
import json

def validate_json_payload(raw: str, field_name: str) -> dict:
    """Parse JSON with helpful error messages for common agent mistakes."""
    try:
        return json.loads(raw)
    except json.JSONDecodeError as e:
        # Common agent mistakes
        if "Expecting ',' delimiter" in str(e):
            hint = "Check for trailing commas or missing commas between fields."
        elif "Expecting property name" in str(e):
            hint = "Check for trailing commas after the last property."
        elif "Expecting value" in str(e):
            hint = "Check for empty values or missing quotes around strings."
        else:
            hint = "Ensure the JSON is valid."

        raise ValueError(
            f"Invalid JSON in '{field_name}' at line {e.lineno}, col {e.colno}: "
            f"{e.msg}. {hint}"
        ) from e
```

## Testing Agent-Specific Failures

Fuzz your inputs with the kinds of mistakes agents make:

```python
AGENT_FUZZ_INPUTS = {
    "path_traversal": [
        "../../etc/passwd",
        "../../../.ssh/id_rsa",
        "..%2F..%2Fetc%2Fpasswd",
        "/absolute/path/outside/sandbox",
    ],
    "embedded_params": [
        "abc123?fields=name",
        "resource_id#section",
        "id&extra=param",
    ],
    "double_encoding": [
        "%2e%2e%2f%2e%2e%2f",
        "hello%20world",
        "file%2Fname",
    ],
    "control_chars": [
        "normal\x00text",
        "line\x08break",
        "tab\x1bescaped",
    ],
    "injection": [
        "'; DROP TABLE users; --",
        "$(rm -rf /)",
        "`whoami`",
        "| cat /etc/passwd",
    ],
}

def test_input_hardening(cli_function, fuzz_inputs):
    """Verify all fuzz inputs are rejected, not processed."""
    for category, inputs in fuzz_inputs.items():
        for bad_input in inputs:
            try:
                cli_function(bad_input)
                print(f"FAIL: {category} input '{bad_input}' was accepted")
            except (ValueError, ValidationError):
                print(f"PASS: {category} input '{bad_input}' was rejected")
```

### What to Test

- Every user-supplied string parameter against all fuzz categories
- Resource IDs against embedded params and double encoding
- File paths against traversal attacks
- JSON payloads against malformed JSON with helpful errors
- `--dry-run` should catch all validation errors before API calls
