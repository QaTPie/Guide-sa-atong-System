# Phoenix Rising — Code Guide 📘

A plain-English walkthrough of every file, written for your **debug exam**.

If you understand this document, you can answer questions about the code *and*
find mistakes in it.

---

## 1. What this project is

A campus emergency reporting site for the **PHOENIX RISING** system.

There are **6 pages**:

| Page | File | Job |
|---|---|---|
| Login | `index.php` | Sign in with email + password |
| Sign Up | `signup.php` | Create a new account |
| Forgot Password | `forgot_password.php` | Set a new password |
| Dashboard | `dashboard.php` | The main screen (private) |
| Logout | `logout.php` | Sign out |
| — | `validation.php` | All the form rules |
| — | `accounts.php` | Reads/writes the account file |

**No database. No framework. No Composer.** Just PHP, HTML, CSS and a little
JavaScript. That is on purpose — it is easy to read.

---

## 2. How to run it

Put the folder in `htdocs`, start **Apache** in XAMPP, then open:

```
http://localhost/phoenix-rising/index.php
```

or, with PHP on your PATH:

```bash
php -S localhost:8000
# then open http://localhost:8000/index.php
```

> ⚠️ **VS Code "Live Server" will NOT work.** It only serves static files, so
> you see a file list or raw code instead of the running site.

---

## 3. How PHP works here (the single most important idea)

A web page in this project is **one file that runs top to bottom every time it
is opened**:

```
Browser asks for index.php
        │
        ▼
PHP runs the top of the file  (the <?php ... ?> part)
        │
        ├── Is this a POST (form submitted)?
        │      YES -> read, validate, maybe redirect
        │      NO  -> skip, just show the form
        │
        ▼
PHP sends the HTML below it back to the browser
        │
        ▼
Browser shows the page
```

**The page talks to itself.** When you press Login, the form posts back to
`index.php` — the same file — which re-runs and either shows errors or
redirects you.

### How the page knows the form was submitted

```php
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // the form was submitted
}
```

- `$_SERVER` — information about the request
- `REQUEST_METHOD` — `GET` (normal visit) or `POST` (form submitted)
- `$_POST` — the values sent by the form, e.g. `$_POST['email']`

---

## 4. The login flow, step by step

```
1. User opens index.php
   -> not a POST -> skip the whole PHP block -> show the empty form

2. User types ana@example.com / Campus2026 and presses LOGIN
   -> form POSTs back to index.php

3. index.php now runs the PHP block:
   a. read   : $oldEmail = trim($_POST['email'] ?? '')
   b. validate: required? valid email? password has a number?
   c. if every field passed:
        accounts_verify() -> look up the account, check the password hash
   d. wrong?  -> $errors['password'] = 'Incorrect email or password.'
      correct? -> $_SESSION['user_email'] = $user['email']
                  header('Location: dashboard.php')
                  exit;
```

**`header('Location: ...')` sends a redirect** (HTTP 302). It must come
**before any HTML output** — and `exit;` right after it stops the rest of the
file from running.

---

## 5. The session — how the site remembers you

HTTP forgets you after every click. PHP uses a **session** to remember.

```php
session_start();                       // <-- must be FIRST, before any HTML

$_SESSION['user_email'] = $user['email'];   // save
$_SESSION['user_name']  = $user['name'];

echo $_SESSION['user_name'];               // read it on another page
```

- `session_start()` creates/resumes the session and sends a `PHPSESSID` cookie
- `$_SESSION` is a special array that survives between pages
- `logout.php` calls `session_unset()` + `session_destroy()`

### The login guard on the dashboard

```php
if (empty($_SESSION['user_email'])) {
    header('Location: index.php');
    exit;
}
```

`empty()` is `true` when the value is missing, `""`, `0` or `null`. So if nobody
is logged in, the dashboard kicks you back to the login page. **This one `if`
is what makes the dashboard private.**

---

## 6. Validation — the instructor's checklist

All the rules live in **`validation.php`**, so each rule is written only once.

### The core convention

Every function returns a **string**:

| Return value | Meaning |
|---|---|
| `''` (empty string) | ✅ the input is fine |
| `'Name is required.'` | ❌ that is the message to display |

So a page can just do:

```php
$errors['email'] = required_field($oldEmail, 'Email');

if ($errors['email'] === '') {          // still OK? then check the format
    $errors['email'] = validate_email($oldEmail, 'Email');
}
```

The second check only runs when the first one passed — otherwise an empty box
would show **two** errors at once.

### The four checklist items

| # | Checklist item | Function | Where it's used |
|---|---|---|---|
| 1 | **Required Field Validation** | `required_field()` | every field, all 3 forms |
| 2 | **Validating Text Input** | `validate_text()` | `name` on Sign Up |
| 3 | **Validating Numbers** | `validate_password_has_number()` | password fields |
| 4 | **Validating Email Addresses** | `validate_email()` | all 3 forms |

Extra rules: `validate_password_length()` (8+ characters) and
`validate_password_match()` (the two reset boxes must agree).

### What each rule actually does

```php
// 1. REQUIRED — trim() removes spaces, so "   " also counts as empty
if (trim($value) === '') {
    return $label . ' is required.';
}

// 2. TEXT — \p{L} means "a letter in any language" (so "Peña" works)
if (!preg_match('/^[\p{L}][\p{L} \-.\']*$/u', $value)) {
    return $label . ' must contain letters, spaces, hyphens or apostrophes only.';
}

// 3. NUMBERS — [0-9] means "one character between 0 and 9"
if (!preg_match('/[0-9]/', $value)) {
    return $label . ' must contain at least one number (0-9).';
}

// 4. EMAIL — built-in filter AND an easy-to-read pattern
$passesFilter  = filter_var($value, FILTER_VALIDATE_EMAIL) !== false;
$matchesFormat = preg_match($pattern, $value) === 1;
```

### Why the forms say `novalidate`

```html
<form method="POST" action="" novalidate>
```

`novalidate` switches **off the browser's** own checking, so the
**server-side PHP rules** are the ones being demonstrated. Remove it and the
browser answers first — which is not what the checklist is testing.

### Printing the errors

```php
<?php field_error($errors, 'email'); ?>
```

```php
function field_error(array $errors, string $key): void
{
    if (!empty($errors[$key])) {
        echo '<p class="field-error">' . htmlspecialchars($errors[$key], ENT_QUOTES, 'UTF-8') . '</p>';
    }
}
```

And to put the typed value back into the box:

```php
value="<?= old('email', $oldEmail) ?>"
```

**`htmlspecialchars()` is a security step** — it turns `<` and `>` into safe
text so a user cannot type HTML or JavaScript into a form and have it run.

---

## 7. `accounts.json` and passwords

There is no database. Accounts are saved as JSON text in `accounts.json`:

```json
[
    {
        "id": "b6269bcf11caafef",
        "name": "Ana Cruz",
        "email": "ana@example.com",
        "password": "$2y$10$NCoI0aETpdpAdAuN2V0cr.s3...",
        "created_at": "2026-10-05T08:37:37+02:00"
    }
]
```

### The `password` field is a HASH, not the password

```php
// SAVING (sign-up / reset)
password_hash($password, PASSWORD_DEFAULT);   // -> "$2y$10$..."  (bcrypt)

// CHECKING (login)
password_verify($typedPassword, $storedHash); // -> true or false
```

- A hash **cannot be turned back** into the original password
- `password_verify()` re-hashes what was typed and compares
- **Never** store plain text, and never use `md5()`/`sha1()` for passwords

### Reading and writing the file

```php
$accounts = accounts_load();    // JSON text  -> PHP array
$accounts[] = [ ... ];          // add one
accounts_save($accounts);       // PHP array  -> JSON text
```

Key functions in `accounts.php`:

| Function | Returns |
|---|---|
| `accounts_load()` | every account as an array |
| `accounts_save($accounts)` | writes the file |
| `accounts_find_by_email($email)` | the account, or `null` |
| `accounts_email_exists($email)` | `true` / `false` |
| `accounts_create($name,$email,$pass)` | `false` if the email is taken |
| `accounts_update_password($email,$pass)` | `false` if not found |
| `accounts_verify($email,$pass)` | the account on success, or `null` |

> ⚠️ **Do not hand-edit `accounts.json`.** One wrong comma makes `json_decode()`
> return `null`, and `accounts_load()` then returns `[]` — so every account
> silently "disappears".

---

## 8. 🔧 DEBUG CHEAT SHEET

### 8.1 Turn error messages ON

By default PHP hides errors. In **XAMPP**, edit `php.ini`:

```ini
display_errors = On
error_reporting = E_ALL
```

…or temporarily at the very top of a file (before any HTML!):

```php
ini_set('display_errors', 1);
error_reporting(E_ALL);
```

**Also useful** — see exactly what the form sent:

```php
echo '<pre>';
print_r($_POST);
echo '</pre>';
exit;          // stop here so you can read it
```

### 8.2 Common PHP errors and what they mean

| Message | What it really means | Fix |
|---|---|---|
| `Parse error: syntax error, unexpected ...` | A `;`, `)`, `}` or quote is missing | Check the **line before** the one reported |
| `Warning: Undefined array key "email"` | You read `$_POST['email']` when it wasn't sent | Use `$_POST['email'] ?? ''` |
| `Warning: Undefined variable $x` | You used `$x` before giving it a value | Set `$x = '';` first |
| `Fatal error: Call to undefined function validate_email()` | Filename typo, or the `require_once` is missing | Add `require_once __DIR__ . '/validation.php';` |
| `Warning: include(...): Failed to open stream` | Wrong path | Use `__DIR__ . '/file.php'` |
| `Cannot modify header information - headers already sent` | Something was printed **before** `header()` | Delete the blank line/space before `<?php` or after `?>` |
| `Trying to access array offset on value of type null` | A function returned `null` and you used it like an array | Check for `null` first |
| `Warning: file_put_contents(...): Permission denied` | Folder not writable | Give the folder write permission |
| `Notice: session_start(): ... headers already sent` | HTML was printed before `session_start()` | Move it to the very top |

> **Golden rule:** in PHP, one stray **space or blank line before `<?php`** can
> break every redirect in the file.

### 8.3 Symptom → cause → fix

| You see | Likely cause | Fix |
|---|---|---|
| Page shows **raw PHP code** | Opened from disk, or served by Live Server | Run it through PHP/Apache |
| Page shows a **list of files** | Live Server (port 5500) | Use `php -S` or XAMPP |
| Login **always** says "Incorrect email or password" | Plain password saved instead of a hash, or the JSON is broken, or email case differs | Use `password_hash()`; check `accounts.json`; compare with `strtolower()` |
| **No errors shown** but nothing saves | The `if (count($errors) === 0)` block never runs, or the POST check fails | `print_r($_POST)` to confirm data arrived |
| **Redirect doesn't happen** | `exit;` missing, or output before `header()` | Add `exit;` right after `header()` |
| Dashboard **always** bounces to login | `session_start()` missing on that page, or it's called after output | Put `session_start();` first on *every* page that uses `$_SESSION` |
| Errors **never appear** on the form | You echo `$errors['x']` directly instead of `field_error($errors, 'x')` | Use `field_error()` |
| **All accounts vanished** | Broken JSON in `accounts.json` | Fix the JSON, or delete the file and sign up again |
| Style/pictures missing | Wrong asset path | Use `assets/style.css` (relative, no leading `/`) |
| `validate_email()` rejects a good address | `||` vs `&&`, or the pattern was edited | Both checks must PASS: `if (!$passesFilter || !$matchesFormat)` |

### 8.4 Bugs that are easy to hide in this code

1. **`=` instead of `===`** — `if ($_SERVER['REQUEST_METHOD'] = 'POST')` assigns instead of comparing, and is always true.
2. **Forgetting `exit;`** after `header('Location: ...')` — the rest of the page still runs.
3. **Missing `?? ''`** on `$_POST[...]` — causes "Undefined array key".
4. **Using `__DIR__`** — `require 'validation.php'` works only if the current folder happens to match; `require __DIR__ . '/validation.php'` always works.
5. **Inverted logic** — `if ($errors)` means "there ARE errors"; `if (count($errors) === 0)` means "there are none". Mixing these up makes the form do the opposite.
6. **`array_filter($errors)`** removes the `''` entries — without it, `count($errors)` counts valid fields too.
7. **Uppercase emails** — saving `Ana@x.com` but looking up `ana@x.com` fails unless both use `strtolower()`.
8. **Reference loops** — `foreach ($accounts as &$account)` needs `unset($account);` afterwards.
9. **`session_start()` after output** — breaks the whole session.
10. **`password_verify($hash, $typed)`** — the arguments are the wrong way round; the typed password comes **first**.

### 8.5 How to test every feature (exam rehearsal)

| # | Do this | Expect |
|---|---|---|
| 1 | Submit the login form empty | "Email is required." / "Password is required." |
| 2 | Email `abc` | "must be a valid address" |
| 3 | Sign Up, Name `Ana123` | "must contain letters, spaces, hyphens or apostrophes only." |
| 4 | Password `abcdefgh` | "must contain at least one number (0-9)." |
| 5 | Password `Ab1` | "must be at least 8 characters long." |
| 6 | Repeat password different | "does not match the new password." |
| 7 | Sign up with an email twice | "already exists" |
| 8 | Log in with a wrong password | "Incorrect email or password." |
| 9 | Log in correctly | Redirect to `dashboard.php`, your name in the greeting |
| 10 | Open `dashboard.php` in a new private window | Redirect to `index.php` |
| 11 | Click Log Out | Back to `index.php`; dashboard now blocked |
| 12 | Forgot-password with an unknown email | "No account was found with that email address." |

**Check the syntax of a file without running it:**

```bash
php -l index.php        # "No syntax errors detected"
```

**Test a page from the terminal:**

```bash
curl.exe -s -i http://localhost:8000/dashboard.php
# look for: HTTP/1.1 302 Found  +  Location: index.php
```

---

## 9. 🧪 Practice: find the bug

Cover the answer, then work out what is wrong.

**1.**
```php
if ($_SERVER['REQUEST_METHOD'] = 'POST') {
```

<details><summary>Answer</summary>
Single `=` **assigns** `'POST'` to `REQUEST_METHOD` and the condition is always
true. It must be `===`.
</details>

**2.**
```php
$errors['email'] = required_field($email, 'Email');
$errors['email'] = validate_email($email, 'Email');
```

<details><summary>Answer</summary>
The second line overwrites the first. An empty field now shows the *format*
error instead of "is required". The second check must be wrapped in
`if ($errors['email'] === '') { ... }`.
</details>

**3.**
```php
header('Location: dashboard.php');

// ... 40 more lines of PHP ...
```

<details><summary>Answer</summary>
No `exit;`. The redirect header is sent but the rest of the script still runs.
Always follow `header('Location: ...')` with `exit;`.
</details>

**4.**
```php
<?
session_start();
```
*(there is a blank line above the `<?`)*

<details><summary>Answer</summary>
That blank line is output, so `session_start()` and any later `header()` fail
with "headers already sent". The file must start with `<?php` on line 1, with
nothing — not even a space — before it.
</details>

**5.**
```php
$accounts[] = [
    'email'    => $email,
    'password' => $password,
];
```

<details><summary>Answer</summary>
The plain password is stored. It must be
`password_hash($password, PASSWORD_DEFAULT)`, and `accounts_verify()` must use
`password_verify()`.
</details>

**6.**
```php
$user = accounts_find_by_email($email);
echo $user['name'];
```

<details><summary>Answer</summary>
`accounts_find_by_email()` returns `null` when the email is unknown, so
`$user['name']` triggers "Trying to access array offset on value of type null".
Check for `null` first.
</details>

**7.**
```php
foreach ($accounts as &$account) {
    $account['password'] = password_hash($newPass, PASSWORD_DEFAULT);
}
```

<details><summary>Answer</summary>
The reference `$account` is never unset. Add `unset($account);` after the loop —
otherwise the last element stays linked to `$account` and the next loop
silently corrupts your array.
</details>

**8.**
```php
echo '<p class="field-error">' . $errors['email'] . '</p>';
```

<details><summary>Answer</summary>
Two problems: it prints even when there is no error (empty `<p>`), and it does
not escape the text. Use
`field_error($errors, 'email');` — it checks `!empty()` **and** calls
`htmlspecialchars()`.
</details>

**9.**
```php
if (!$passesFilter && !$matchesFormat) {
    return 'must be a valid address...';
}
```

<details><summary>Answer</summary>
`&&` means "both failed". It should be `||` — "either check failed". With `&&`
an address can fail one check and still be accepted.
</details>

**10.**
```php
$errors = array_filter($errors);

if ($errors) {
    // ... save the account ...
}
```

<details><summary>Answer</summary>
Inverted. `if ($errors)` means "there ARE errors". Saving must happen only when
there are **none**: `if (count($errors) === 0) { ... }`.
</details>

---

## 10. Glossary

| Thing | Meaning |
|---|---|
| `$_POST` | values sent by a submitted form |
| `$_GET` | values in the URL (`page.php?id=3`) |
| `$_SESSION` | values that survive between pages |
| `$_SERVER` | info about the request (`REQUEST_METHOD`, etc.) |
| `trim($s)` | remove spaces from both ends |
| `isset($x)` | true when the variable exists and isn't `null` |
| `empty($x)` | true when missing, `""`, `0` or `null` |
| `strtolower($s)` | make lowercase |
| `strlen($s)` / `mb_strlen($s)` | length in bytes / in characters |
| `preg_match($pattern, $s)` | does `$s` match the regular expression? |
| `filter_var($s, FILTER_VALIDATE_EMAIL)` | PHP's built-in email check |
| `htmlspecialchars($s)` | escape HTML — protects against XSS |
| `header('Location: x.php')` | send a redirect (must be before output) |
| `exit;` | stop the script immediately |
| `require_once` | load another file, only once |
| `__DIR__` | the folder this file lives in |
| `json_decode($j, true)` | JSON text → PHP array |
| `json_encode($a)` | PHP array → JSON text |
| `file_get_contents` / `file_put_contents` | read / write a file |
| `password_hash` / `password_verify` | make / check a password hash |
| `array_filter($a)` | remove empty values |
| `count($a)` | how many items |
| `explode(' ', $s)` | split a string into an array |
| `implode(', ', $a)` | join an array into a string |
| `date('c')` | date in ISO format, e.g. `2026-10-05T08:37:37+02:00` |

---

## 11. Questions your instructor may ask

**Q: Why is there no database?**
The project brief only needed login and sign-up, so accounts are kept in a JSON
file — fewer setup steps and everything is visible in one place.

**Q: Why `novalidate` on the form?**
So the *server-side PHP* rules do the validating. The checklist is about PHP
validation, not the browser's.

**Q: Why `password_hash()` instead of saving the password?**
If the file leaks, hashes cannot be reversed into the real passwords. It is also
deliberately slow, which makes guessing attacks impractical.

**Q: What does `exit;` do after `header()`?**
`header()` only *queues* the redirect. Without `exit;` the script keeps running
and may print content or run more database code.

**Q: Why is `$_SESSION` used instead of a normal variable?**
Normal variables are forgotten when the page finishes. `$_SESSION` is stored on
the server and tied to the visitor's cookie, so the next page knows who it is.

**Q: How is the dashboard protected?**
The `empty($_SESSION['user_email'])` check at the top of `dashboard.php` redirects
anyone who isn't logged in.

**Q: What does `??` mean?**
"Use the left side if it exists, otherwise the right side":
`$_POST['email'] ?? ''`.

**Q: Why do the validators return `''` on success?**
So the pages can store them in `$errors` directly and then simply ask
"is `$errors` empty?".

---

## 12. Where things are

| I want to change… | Edit… |
|---|---|
| A validation message | `validation.php` |
| How a password is stored | `accounts.php` |
| The redirect after login | `index.php` (bottom of the PHP block) |
| Which page is private | `dashboard.php` (top) |
| Colours of the login/sign-up pages | `assets/style.css` |
| Colours of the dashboard | `assets/dashboard.css` |
| The show/hide password button | `assets/password-toggle.js` |
| The HTML of a page | the bottom half of that page's `.php` file |
