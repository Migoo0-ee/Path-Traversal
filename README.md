# Path Traversal

A collection of notes and practical examples for learning **File Path Traversal** vulnerabilities, based on PortSwigger Web Security Academy labs.

## Labs Covered

This repository covers the following six labs:

1. **File Path Traversal, Simple Case**

   * Basic `../` traversal.
   * Accessing `/etc/passwd` through a vulnerable `filename` parameter.

2. **File Path Traversal, Traversal Sequences Blocked with Absolute Path Bypass**

   * Bypassing blocked `../` sequences.
   * Using an absolute path such as `/etc/passwd`.

3. **File Path Traversal, Traversal Sequences Stripped Non-Recursively**

   * Bypassing non-recursive filtering.
   * Using nested traversal sequences such as `....//`.

4. **File Path Traversal, Traversal Sequences Stripped with Superfluous URL-Decode**

   * Using double URL encoding.
   * Example: `..%252f..%252f..%252fetc/passwd`

5. **File Path Traversal, Validation of Start of Path**

   * Bypassing validation that requires the path to start with a specific directory.
   * Example: `/var/www/images/../../../etc/passwd`

6. **File Path Traversal, Validation of File Extension with Null Byte Bypass**

   * Bypassing file-extension validation using a null byte.
   * Example: `../../../etc/passwd%00.png`

## Mitigation

The repository also contains notes about preventing path traversal vulnerabilities, including:

* Avoiding user-controlled filesystem paths whenever possible.
* Using allowlists for permitted files.
* Canonicalizing/resolving paths before accessing them.
* Verifying that the resolved path remains inside the intended base directory.
* Examples of vulnerable and mitigated implementations using **Django** and **PHP**.

## Example

A vulnerable application might directly concatenate user input with a filesystem path:

```python
file_path = os.path.join(BASE_DIR, user_input)
```

A safer implementation resolves the final path and verifies that it remains inside the intended directory.

## Purpose

This repository is for **educational purposes**, personal security study, and practicing web application security in authorized environments such as PortSwigger Web Security Academy labs.

## Topics

* File Path Traversal
* Directory Traversal
* Absolute Path Bypass
* Traversal Filter Bypass
* URL Encoding
* Double URL Encoding
* Path Validation
* Null Byte Bypass
* Secure File Handling
* Django
* PHP

## Disclaimer

Use these techniques only against applications and systems that you own or have explicit permission to test.
