# Web Application Security Vulnerability Report

## 📝 Overview

This project involves identifying and fixing security vulnerabilities within a provided web application hosted on GitLab (`image-rocket`, https://gitlab.utwente.nl/software-security/image-rocket). The vulnerabilities are expected to align with the OWASP Top 10 list discussed in lectures. The task can be approached through manual code review or by utilizing automated tools like OWASP ZAP. Using a specialized distribution like Kali Linux is suggested for access to pre-installed security tools. Additionally, the project requires writing a restrictive Content Security Policy (CSP) for the application, potentially involving minor application modifications to achieve stricter security rules.

---

##  Vulnerabilities Found and Fixed

### 1. Cross-Site Scripting (XSS) Vulnerability 💉

* **Location**: User input fields for comments and usernames.
* **Cause**:
    1.  User-generated content (comments, usernames) was stored in the database without proper sanitization.
    2.  This stored data, potentially containing malicious scripts (e.g., `<script>alert('test');</script>`), was rendered directly onto web pages without escaping special HTML characters.
* **Fix**:
    1.  **Sanitization**: Used the `ammonia` Rust crate to clean user inputs (`comment.comment` and `comment.user_name`) *before* inserting them into the database. `ammonia::clean()` removes or escapes dangerous elements like `<script>`, `<iframe>`, and JavaScript event handlers (`onclick`, etc.).
    2.  **Safe Database Insertion**: Inserted the *sanitized* content into the database using parameterized queries, ensuring only safe data is stored.

---

### 2. Time-Based Blind SQL Injection Vulnerability ⏳

* **Location**: The `q` parameter in the `/search` endpoint.
* **Cause**: The application constructed SQL queries dynamically by directly embedding user input from the `q` parameter into the SQL string using `format!`.
* **Identification**: Detected using the `sqlmap` tool. Injecting payloads like `test') AND 9037=(SELECT 9037 FROM PG_SLEEP(5)) AND ('afxH'='afxH` forced the PostgreSQL database to delay its response by 5 seconds, confirming the vulnerability.
* **Fix**: Replaced the dynamic query construction with **parameterized queries**. User input is now treated strictly as data, not executable SQL code, preventing injection attacks.

---

### 3. Command Injection Vulnerability 💻

* **Location**: Image resizing functionality triggered during file upload.
* **Cause**: User-provided file paths (`some_path`, derived from uploaded filenames) were directly inserted into a shell command string that called the external `convert` (ImageMagick) utility. An attacker could craft a filename like `"image.jpg; rm -rf /"` to execute arbitrary commands on the server.
* **Fix**:
    1.  **Eliminated Shell Command**: The external `convert` command was completely removed.
    2.  **Used Native Library**: Image resizing is now handled safely within the Rust application using the `image` crate.
    3.  **Sanitized File Paths**: Standard Rust libraries (`std::path`) are used to handle file paths securely.

---

### 4. Format String Error Vulnerability 🔡

* **Location**: Username field associated with comments.
* **Cause**: The application allowed potentially problematic characters (like `'` and `%`) in usernames without proper validation or sanitization. While not a direct format string *attack* in this context (like in C's `printf`), these characters could cause errors or unexpected behavior if passed to backend functions that perform string formatting or processing. Detected using the ZAP tool.
* **Fix**:
    1.  **Input Validation**: Implemented a `sanitize_for_username` function using the `regex` crate.
    2.  **Strict Regex**: This function uses the regular expression `r"^[a-zA-Z0-9_\-]+$"` to ensure usernames contain *only* alphanumeric characters, underscores (`_`), and hyphens (`-`). Any username containing other characters is rejected or defaulted to "unknown_user". This prevents problematic characters from being stored or processed.

---

## 🔒 Secure File Upload Design

Several improvements were made to the file upload functionality (`upload_post` function) to enhance security and robustness:

1.  **Handled File Size Errors**: Correctly handled the `Result` returned by `form.file.len()` using a `match` statement to prevent potential crashes if the file size couldn't be determined. Files with undefined sizes are now rejected.
2.  **Validated File Content Type**: Implemented validation based on the file's *actual content*, not just its extension. Uses `image::guess_format` to read the file data and determine the image format. Only **JPEG** and **PNG** formats are accepted; all other file types (including potentially malicious files disguised as images) are rejected.
3.  **Imported Missing Dependency**: Added `use image::ImageFormat;` to resolve compilation errors related to file type validation.

---

## 🛡️ Content Security Policy (CSP) Implementation

A Content Security Policy was implemented to mitigate XSS and data injection attacks by restricting the sources from which resources (scripts, styles, images, etc.) can be loaded.

* **Implementation**: A custom **Rocket Fairing** (`CSPFairing`) was created. This fairing intercepts every HTTP response and adds the `Content-Security-Policy` header.
* **Policy Directives**: The implemented policy includes:
    * `default-src 'self'`: Restricts loading most resources to the same origin by default.
    * `script-src 'self' https://code.jquery.com https://cdnjs.cloudflare.com https://stackpath.bootstrapcdn.com`: Allows scripts only from the same origin and specific trusted CDNs (jQuery, Cloudflare, Bootstrap).
    * `style-src 'self' https://stackpath.bootstrapcdn.com`: Allows CSS from the origin and Bootstrap CDN.
    * `font-src 'self' https://stackpath.bootstrapcdn.com`: Allows fonts from the origin and Bootstrap CDN.
    * `img-src 'self'`: Allows images only from the same origin.
    * `object-src 'none'`: Disallows plugins like Flash or Java applets.
    * `form-action 'self'`: Allows HTML forms to submit data only to the same origin.
    * `frame-ancestors 'none'`: Prevents the site from being embedded in `<iframe>` or `<frame>` elements on other sites (clickjacking protection).
* **Rationale**: The policy was tailored based on the application's actual needs, allowing necessary external libraries (jQuery, Bootstrap) while blocking potentially malicious content from untrusted sources.
