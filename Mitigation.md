# Preventing Path Traversal Vulnerabilities

The most effective way to prevent path traversal vulnerabilities is to avoid passing user-supplied input directly to filesystem APIs whenever possible.

If user input must be used to access a file, use **defense in depth**:

1. **Validate the input** before processing it. Ideally, use a whitelist of permitted values.
    
2. **Resolve the final path to its canonical/absolute form** and verify that it remains inside the intended base directory.
    

---

# Django

## Vulnerable Code

The following example directly uses the user-controlled `filename` parameter to construct a file path:

```python
from django.http import FileResponse
from django.conf import settings
import os

def download_file(request):
    filename = request.GET.get("filename")

    file_path = os.path.join(settings.MEDIA_ROOT, filename)

    return FileResponse(open(file_path, "rb"))
```

An attacker could provide:

```text
?filename=../../../../etc/passwd
```

Depending on the filesystem and application configuration, this could cause the application to read a file outside the intended directory.

### Why is it vulnerable?

The application trusts `filename` and passes it directly to the filesystem without verifying that the resulting path stays inside `MEDIA_ROOT`.

---

## Mitigated Code

A safer approach is to resolve the path and verify that it remains inside the intended base directory:

```python
from django.http import FileResponse, Http404
from django.conf import settings
from pathlib import Path

def download_file(request):
    filename = request.GET.get("filename", "")

    base_dir = Path(settings.MEDIA_ROOT).resolve()
    file_path = (base_dir / filename).resolve()

    try:
        file_path.relative_to(base_dir)
    except ValueError:
        raise Http404("Invalid file path")

    if not file_path.is_file():
        raise Http404("File not found")

    return FileResponse(open(, "rfile_pathb"))
```

### Why is this safer?

The application:

1. Resolves the base directory.
    
2. Resolves the user-supplied path.
    
3. Checks whether the resulting path is still inside the allowed directory.
    
4. Rejects the request if the path escapes the base directory.
    

For example:

```text
Base directory:
/var/www/images

User input:
../../etc/passwd

Resolved path:
/var/etc/passwd
```

Because the resolved path is outside `/var/www/images`, the request is rejected.

### Even Better: Use a Whitelist

If the application only needs to serve specific files, don't accept arbitrary filenames at all.

```python
ALLOWED_FILES = {
    "logo": "logo.png",
    "profile": "profile.png",
    "banner": "banner.jpg",
}

def download_file(request):
    file_key = request.GET.get("file")

    filename = ALLOWED_FILES.get(file_key)

    if not filename:
        raise Http404("Invalid file")

    file_path = Path(settings.MEDIA_ROOT) / filename

    return FileResponse(open(file_path, "rb"))
```

Here, the user does not control the actual filesystem path. They can only choose from predefined values.

---

# PHP

## Vulnerable Code

A vulnerable PHP implementation might look like this:

```php
<?php

$filename = $_GET['filename'];

$baseDir = "/var/www/images/";
$filePath = $baseDir . $filename;

readfile($filePath);
?>
```

An attacker could request:

```text
?filename=../../../../etc/passwd
```

The application would construct:

```text
/var/www/images/../../../../etc/passwd
```

which can resolve outside the intended directory.

### Why is it vulnerable?

The application concatenates user-controlled input directly with a filesystem path without validating the final path.

---

## Mitigated Code

PHP provides `realpath()` to resolve a path to its canonical absolute path.

```php
<?php

$filename = $_GET['filename'];

$baseDir = realpath("/var/www/images");

$filePath = realpath($baseDir . DIRECTORY_SEPARATOR . $filename);

if ($filePath === false) {
    http_response_code(404);
    exit("File not found");
}

if (strncmp($filePath, $baseDir . DIRECTORY_SEPARATOR, strlen($baseDir . DIRECTORY_SEPARATOR)) !== 0) {
    http_response_code(403);
    exit("Invalid file path");
}

readfile($filePath);
?>
```

### How it works

Suppose:

```text
Base directory:
/var/www/images
```

The attacker submits:

```text
../../etc/passwd
```

After canonicalization, the resulting path is outside the allowed directory.

The check detects this and rejects the request.

---

## PHP — Whitelist Approach

If only a small number of files are required, a whitelist is preferable:

```php
<?php

$allowedFiles = [
    "logo" => "logo.png",
    "banner" => "banner.jpg",
    "profile" => "profile.png"
];

$fileKey = $_GET['file'] ?? null;

if (!isset($allowedFiles[$fileKey])) {
    http_response_code(404);
    exit("Invalid file");
}

$filePath = "/var/www/images/" . $allowedFiles[$fileKey];

readfile($filePath);
?>
```

The attacker cannot provide:

```text
../../etc/passwd
```

because the application never uses arbitrary user input as the filename. The user can only select one of the predefined files.

---

# Key Takeaway

There are two important defensive approaches:

### 1. Whitelist

Best when possible:

```text
User input → Allowed value → Known file
```

Example:

```text
?file=logo
        ↓
logo → logo.png
```

### 2. Canonicalize + Verify

When arbitrary filenames are required:

```text
User input
    ↓
Build path
    ↓
Canonicalize / Resolve
    ↓
Check it starts inside the allowed base directory
    ↓
Process the file
```

The important point is that **checking the raw user input is not enough**. The security check should be performed on the **resolved/canonical path**, because traversal sequences can change what the final filesystem path actually points to.