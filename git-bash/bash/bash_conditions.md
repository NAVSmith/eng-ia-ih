# Bash Conditional Operators

This guide provides a comprehensive list of conditional operators used in Bash scripting, primarily within `[ ... ]` (test) and `[[ ... ]]` constructs.

**Note on `[` vs `[[`:**

*   `[ ... ]` (test): The original, POSIX-compliant test command. It's more portable but can be tricky with word splitting and filename expansion.
*   `[[ ... ]]`: An extended and more powerful version available in Bash (and other modern shells). It offers more features, is safer against common errors, and should be preferred for Bash-specific scripts.

---

## 1. Integer Comparison Operators

These operators are used for comparing numerical values.

| Operator | Description              | Example (`a=10`, `b=20`)                |
| :------- | :----------------------- | :-------------------------------------- |
| `-eq`    | **E**qual to             | `if [ "$a" -eq 10 ]; then ... fi`       |
| `-ne`    | **N**ot **e**qual to     | `if [ "$a" -ne "$b" ]; then ... fi`     |
| `-gt`    | **G**reater **t**han     | `if [ "$b" -gt "$a" ]; then ... fi`     |
| `-ge`    | **G**reater than or **e**qual to | `if [ "$b" -ge 20 ]; then ... fi`       |
| `-lt`    | **L**ess **t**han        | `if [ "$a" -lt "$b" ]; then ... fi`     |
| `-le`    | **L**ess than or **e**qual to  | `if [ "$a" -le 10 ]; then ... fi`       |

**Example:**
```bash
count=10
if [ "$count" -eq 10 ]; then
  echo "The count is 10."
fi
```

---

## 2. String Comparison Operators

These operators are used for comparing strings. Always quote your variables to prevent errors with empty strings or strings containing spaces.

| Operator | Description                 | Example (`s1="abc"`, `s2="xyz"`)         |
| :------- | :-------------------------- | :--------------------------------------- |
| `=` or `==` | Equal to                    | `if [ "$s1" = "abc" ]; then ... fi`       |
| `!=`     | Not equal to                | `if [ "$s1" != "$s2" ]; then ... fi`      |
| `-z`     | String is empty (zero length) | `if [ -z "$empty_var" ]; then ... fi`    |
| `-n`     | String is not empty         | `if [ -n "$s1" ]; then ... fi`           |
| `<`      | Less than (ASCII order)     | `if [[ "$s1" < "$s2" ]]; then ... fi`    |
| `>`      | Greater than (ASCII order)  | `if [[ "$s2" > "$s1" ]]; then ... fi`    |

**Important:**
*   `==` is a Bash-specific alias for `=`. For POSIX compliance, use `=`.
*   `<` and `>` **must** be used inside `[[ ... ]]` for string comparison to avoid being interpreted as redirection operators.

**Example:**
```bash
name="Alice"
if [ "$name" = "Alice" ]; then
  echo "Hello, Alice!"
fi

empty_string=""
if [ -z "$empty_string" ]; then
  echo "The string is empty."
fi
```

---

## 3. File and Directory Conditionals

These operators check the status of files and directories on the filesystem.

| Operator | Description                               | Example (`file="/path/to/file.txt"`) |
| :------- | :---------------------------------------- | :----------------------------------- |
| `-e`     | File or directory **e**xists              | `if [ -e "$file" ]; then ... fi`     |
| `-f`     | Path exists and is a regular **f**ile     | `if [ -f "$file" ]; then ... fi`     |
| `-d`     | Path exists and is a **d**irectory        | `if [ -d "/home" ]; then ... fi`     |
| `-r`     | File is **r**eadable by the current user  | `if [ -r "$file" ]; then ... fi`     |
| `-w`     | File is **w**ritable by the current user  | `if [ -w "$file" ]; then ... fi`     |
| `-x`     | File is e**x**ecutable by the current user| `if [ -x "/bin/ls" ]; then ... fi`   |
| `-s`     | File is not empty (has a **s**ize > 0)    | `if [ -s "$file" ]; then ... fi`     |
| `-L` or `-h` | Path exists and is a symbolic **l**ink    | `if [ -L "/path/to/symlink" ]; then ... fi` |
| `file1 -nt file2` | `file1` is **n**ewer **t**han `file2` | `if [ "a.txt" -nt "b.txt" ]; then ... fi` |
| `file1 -ot file2` | `file1` is **o**lder **t**han `file2` | `if [ "a.txt" -ot "b.txt" ]; then ... fi` |

**Example:**
```bash
config_file="~/.my_app.conf"
if [ -f "$config_file" ]; then
  echo "Config file found. Reading settings..."
else
  echo "Config file not found."
fi
```

---

## 4. Logical Operators

These combine multiple conditions.

| Operator | Description | `[ ]` Example | `[[ ]]` Example |
| :--- | :--- | :--- | :--- |
| `!` | **NOT** - Inverts the result of the condition | `if [ ! -f "a.txt" ]; then ... fi` | `if [[ ! -f "a.txt" ]]; then ... fi` |
| `-a` / `&&` | **AND** - True if both conditions are true | `if [ "$a" -gt 0 -a "$a" -lt 10 ]; then ... fi` | `if [[ "$a" -gt 0 && "$a" -lt 10 ]]; then ... fi` |
| `-o` / `||` | **OR** - True if either condition is true | `if [ "$ans" = "y" -o "$ans" = "Y" ]; then ... fi` | `if [[ "$ans" == "y" || "$ans" == "Y" ]]; then ... fi` |

**Important:**
*   The `&&` and `||` operators are recommended and should be used within `[[ ... ]]`.
*   The `-a` and `-o` operators are for use with `[ ... ]` and are considered deprecated due to their tricky parsing rules.

**Example:**
```bash
age=25
has_ticket=true

if [[ "$age" -ge 18 && "$has_ticket" == true ]]; then
  echo "Access granted."
else
  echo "Access denied."
fi
```
