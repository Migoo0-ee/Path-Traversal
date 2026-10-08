# LFI, LFD, RFI, Path Traversal & RCE

---

## 1. LFI — Local File Inclusion

### What is LFI?

**Local File Inclusion (LFI)** is a vulnerability that occurs when an application uses user-controlled input in a file inclusion operation without properly validating or restricting the supplied path.

A vulnerable application may contain logic similar to:

```text
?page=about.php
```

where the value of `page` influences which local resource the application includes.

### How does it work?

The attacker attempts to manipulate the file parameter so that the application accesses a different local resource than intended.

A common technique is **Path Traversal**, using sequences such as:

```text
../
```

For example:

```text
?page=../../../../etc/passwd
```

Whether this works depends on the application's implementation, filtering, permissions, and operating system.

### Important

**LFI does NOT automatically mean RCE.**

LFI can potentially lead to:

- Local File Disclosure
    
- Source-code disclosure
    
- Code execution under specific conditions
    

---

# 2. LFD — Local File Disclosure

### What is LFD?

**Local File Disclosure (LFD)** is the ability to read or disclose files that should not be accessible to the user.

The important distinction is:

- **LFI** → the application includes a local resource.
    
- **LFD** → sensitive local file contents are disclosed.
    
- **Path Traversal** → manipulation of a file path to access a location outside the intended directory.
    

They can overlap, but they are **not the same vulnerability**.

### Where can LFI/LFD appear?

Look for parameters whose values appear to reference files or resources, for example:

```text
?page=
?file=
?filename=
?folder=
?document=
?path=
```

Common application features where these issues may appear include:

- Dynamic page/resource loading
    
- Image loaders
    
- Document viewers
    
- PDF viewers
    
- Download functionality
    
- Template/resource loading
    

> Parameter names alone do not prove a vulnerability. The important thing is how the application uses the value.

---

# 3. Path Traversal

### What is Path Traversal?

**Path Traversal** is a vulnerability/technique that allows an attacker to manipulate a file path and access resources outside the application's intended directory.

The classic traversal sequence is:

```text
../
```

For example, an application may expect:

```text
?page=index.php
```

and the attacker may attempt to traverse upward:

```text
?page=../../../../etc/passwd
```

The exact number of `../` sequences required depends on the application's directory structure.

### Important distinction

`../` is the **traversal sequence**.

**Path Traversal** is the technique/vulnerability involving manipulation of the path to escape the intended location.

Path Traversal can result in **LFD**, but it does not necessarily involve LFI.

### Mental model

```text
Path Traversal
      ↓
Access outside intended directory
      ↓
Potential file disclosure (LFD)
```

---

# 4. PHP Stream Wrappers

### What are PHP wrappers?

PHP provides **stream wrappers**, which allow PHP functions to interact with different types of resources and data streams.

Examples include:

```text
file://
php://
data://
http://
```

Some wrappers become security-relevant when an application allows attacker-controlled input to reach a file operation.

---

## `file://`

The `file://` wrapper can reference local filesystem resources.

Example concept:

```text
file://path/to/file
```

Its usefulness depends on the vulnerable application's behavior and PHP configuration.

---

## `php://filter`

`php://filter` can apply a filter to a stream before the application receives the content.

One important security use case is **source-code disclosure**.

For example:

```text
?page=php://filter/read=convert.base64-encode/resource=index.php
```

The idea is:

```text
PHP source
    ↓
Base64 encoding
    ↓
Encoded output
    ↓
Decode it to inspect the source
```

### Why is Base64 useful here?

Normally, if PHP includes a PHP file, the PHP interpreter may execute the PHP code rather than displaying the source.

Encoding the stream first can allow the source to be returned as encoded data instead.

Therefore:

```text
php://filter
      ↓
Source transformation
      ↓
Source-code disclosure
```

**This is primarily a source-disclosure technique, not automatically an RCE technique.**

---

## `expect://`

The `expect` wrapper can potentially provide command execution in configurations where it is available.

However:

- It is **not enabled by default** in normal PHP installations.
    
- Its availability depends on the environment.
    
- You should not assume that `expect://` exists on a target.
    

Therefore:

```text
expect:// ≠ guaranteed RCE
```

---

# 5. LFI → Source Disclosure

One important scenario is when an LFI vulnerability allows PHP files to be included.

Simply requesting a PHP source file may cause the server to interpret it rather than disclose its source.

A filtering mechanism such as:

```text
php://filter
```

can sometimes transform the contents before they are processed.

Conceptually:

```text
LFI
 ↓
php://filter
 ↓
Transform source
 ↓
Source-code disclosure
```

This can reveal:

- Application logic
    
- Database configuration
    
- API keys/secrets accidentally stored in source
    
- Other useful information for further security testing
    

---

# 6. LFI → RCE

### Important concept

Having LFI does **not** automatically give you Remote Code Execution.

For LFI → RCE to occur, there generally needs to be a way for attacker-controlled content to become something that the PHP interpreter executes through the inclusion mechanism.

Think about the required conditions:

```text
LFI
 ↓
Can I control the included content?
 ↓
Can the application include that content?
 ↓
Will PHP interpret it as executable PHP?
 ↓
Potential RCE
```

---

## Common LFI → RCE Conditions

### 1. Attacker-controlled uploaded content

If an application allows an attacker to upload content and that content can later be included by the vulnerable file parameter, the included content may become an execution primitive depending on how the server handles it.

The important part is not simply:

> "I can upload a file."

The important question is:

> "Can the vulnerable inclusion mechanism reach attacker-controlled content and cause it to be interpreted as PHP?"

---

### 2. PHP Session Files

PHP applications can store session data in files.

A common example on some systems is:

```text
/var/lib/php/sessions/
```

or another configured session directory.

The exact location is **configuration-dependent** and should never be assumed to be universal.

Conceptually:

```text
Attacker-controlled session data
          ↓
Stored in a server-side session file
          ↓
LFI reaches the session file
          ↓
PHP interprets attacker-controlled content
          ↓
Potential RCE
```

The critical requirement is that the application must actually use file-based sessions in a location accessible through the inclusion primitive.

---

### 3. `php://input`

`php://input` provides access to the raw request body in PHP.

It is sometimes discussed in the context of LFI → RCE, but whether it is useful depends heavily on:

- PHP version
    
- PHP configuration
    
- How the vulnerable include operation works
    
- How the request body is processed
    
- Whether the included stream is actually interpreted as PHP
    

Therefore, don't memorize:

```text
php://input = RCE
```

Instead remember:

```text
php://input
    ↓
Raw request body
    ↓
Potentially relevant to some LFI scenarios
    ↓
Only works when the surrounding conditions allow it
```

---

# 7. LFI vs LFD vs Path Traversal

This is the most important distinction to remember.

|Concept|Meaning|
|---|---|
|**Path Traversal**|Manipulating a path to access a location outside the intended directory|
|**LFD**|Disclosure/read access to a local file|
|**LFI**|Application includes a local resource based on attacker-influenced input|
|**RFI**|Application includes a remote resource|
|**RCE**|Attacker achieves execution of commands/code on the target|

They can be related:

```text
Path Traversal ──────→ LFD
                        

LFI ─────────────────→ LFD
 │
 │ under specific conditions
 ↓
RCE
```

But they are **not interchangeable terms**.

---

# 8. Practical Testing Mindset

When you find a parameter such as:

```text
?page=
?file=
?filename=
?path=
```

don't immediately think:

> "This is LFI."

Instead ask:

### Step 1 — What does the parameter control?

Does it control:

- A filename?
    
- A template?
    
- A document?
    
- An image?
    
- A directory?
    
- A download?
    

### Step 2 — Can the path escape its intended location?

Look for path-handling weaknesses and traversal behavior.

### Step 3 — What is the impact?

Can you:

```text
Read a local file?
        ↓
Disclose application source?
        ↓
Include a local resource?
        ↓
Execute attacker-controlled code?
```

### Step 4 — What are the environmental constraints?

Check:

- Operating system
    
- PHP version/configuration
    
- File permissions
    
- Application logic
    
- Upload functionality
    
- Session configuration
    
- Filtering/normalization
    

---

# 9. Core Mental Model

Don't memorize this topic as a list of payloads.

Memorize the relationships:

```text
                ┌──→ LFD
                │
Path Traversal ─┤
                │
                └──→ Access outside intended path


LFI
 │
 ├──→ Local file disclosure
 │
 ├──→ Source-code disclosure
 │
 └──→ RCE
       │
       └── only when additional conditions exist
```

### The key rule

> **LFI is a primitive. RCE is an impact that requires additional conditions.**

That distinction will save you a lot of confusion when you start solving real-world labs.