# Module 7: Advanced Shell Scripting & Automation

A comprehensive technical engineering reference guide covering Bash datatypes, arithmetic, variable attributes, and advanced workflow constructions. This module focuses on building robust, production-ready automation scripts without overcomplicating syntax.

---

## 1. Datatypes in Bash

Unlike compiled languages like Go or object-oriented languages like Python, Bash does not implement strict data types such as integers, booleans, or objects. In Bash, **everything is fundamentally a string**. When you define a number such as `42`, Bash stores it internally as the ASCII character "4" followed by "2". 

However, Bash provides internal evaluation engines that temporarily treat strings as numbers, indexed lists, or associative dictionaries when instructed by specific syntax constructs.

### Architectural Deep Dive: Macro Expansion and Word Splitting

Bash operates as a macro expansion language. When you execute a script line, the interpreter scans the text, substitutes variables, expands wildcards, and splits the line into distinct command arguments based on whitespace (defined by the Internal Field Separator, or `IFS`).

This design leads to two critical behaviors every DevOps engineer must understand:

1. **Whitespace around assignment operators:** In Bash, `VAR="value"` is a variable assignment. However, `VAR = "value"` attempts to execute a command named `VAR` with `=` and `"value"` passed as positional arguments, resulting in a `VAR: command not found` error.
2. **Word splitting on unquoted variables:** If a variable contains spaces, referencing it without double quotes causes Bash to fragment the single string into multiple discrete arguments. Always wrap variable references in double quotes (for example, `"$TARGET_FILE"` or `"${SERVERS[@]}"`) to guarantee argument boundaries remain intact.

---

### 1.1 Strings (The Default)

Because strings represent the default storage model in Bash, declaring them requires no special keywords or formatting. You assign text directly to an identifier. String concatenation is performed by placing variable expansions adjacent to one another.

To prevent ambiguity when concatenating variables with literal text, enclose variable names in curly braces (`${VAR}`). This explicitly marks where the variable identifier begins and ends.

The following script initializes role and identifier variables, then concatenates them into a standardized host identifier:

```bash
# Standard string assignment (no spaces around the '=' sign)
SERVER_ROLE="webserver"
SERVER_ID="01"

# String concatenation using curly braces to delimit variable boundaries
HOSTNAME="${SERVER_ROLE}-${SERVER_ID}"
echo "Deploying to: $HOSTNAME"
# Output: Deploying to: webserver-01
```

---

### 1.2 Numbers & Arithmetic (`$(( ))`)

If you attempt arithmetic using standard variable assignment, such as `TOTAL=1+1`, Bash assigns the literal string `"1+1"`. To force mathematical evaluation, wrap the expression in double-parentheses arithmetic expansion syntax: `$(( expression ))`.

> [!IMPORTANT]
> Bash only supports **integer math**. Decimals (floating-point numbers) are not supported natively and are strictly truncated toward zero. For example, `$(( 10 / 3 ))` evaluates to `3`, not `3.33`.

Bash supports the standard set of integer arithmetic operators:
- Addition: `+`
- Subtraction: `-`
- Multiplication: `*`
- Division: `/` (truncates decimals)
- Modulo: `%` (returns the remainder of integer division)

The modulo operator (`%`) is particularly valuable in DevOps workflows for distributing load across worker pools, calculating batch offsets, or validating odd/even iterations.

The following script demonstrates memory capacity calculations, integer division truncation, modulo remainder extraction, and a compound percentage formula:

```bash
TOTAL_RAM=4096
USED_RAM=1024

# 1. Subtraction (-)
FREE_RAM=$((TOTAL_RAM - USED_RAM))
echo "Free RAM: $FREE_RAM MB"

# 2. Addition (+)
PROJECTED_RAM=$((USED_RAM + 500))
echo "Projected RAM after update: $PROJECTED_RAM MB"

# 3. Multiplication (*)
DOUBLE_CAPACITY=$((TOTAL_RAM * 2))
echo "Doubled Capacity: $DOUBLE_CAPACITY MB"

# 4. Division (/) - Truncates decimals: 4096 / 1000 = 4
NODES=$((TOTAL_RAM / 1000))
echo "Nodes supported: $NODES"

# 5. Modulo (%) - Returns remainder: 4096 % 1000 = 96
REMAINDER=$((TOTAL_RAM % 1000))
echo "Remainder RAM left over: $REMAINDER MB"

# 6. Compound equation for percentage utilization
PERCENT_USED=$(( (USED_RAM * 100) / TOTAL_RAM ))
echo "Memory Utilization: $PERCENT_USED%"
```

---

### 1.3 Lists (Indexed Arrays) & The `declare` Command

When managing groups of infrastructure assets—such as a list of IP addresses, container IDs, or cluster nodes—individual scalar variables become unmanageable. Bash provides Indexed Arrays to maintain ordered, zero-indexed collections of strings.

#### The `declare` Builtin Command

To configure variable attributes explicitly, Bash provides the `declare` builtin. While simple scalar strings do not require `declare`, arrays and typed variables depend on it for correct runtime behavior.

The table below lists the essential `declare` attribute flags used in systems automation:

| Flag | Attribute Description | DevOps Use Case |
| :--- | :--- | :--- |
| `-a` | **Indexed Array** | Ordered lists of hostnames, IP addresses, or ports. |
| `-A` | **Associative Array** | Key-value dictionaries (mapping hostnames to IPs). |
| `-r` | **Read-Only** | Immutable system constants (configuration paths, production ports). |
| `-i` | **Integer** | Forces arithmetic evaluation on assignment without `$(( ))`. |
| `-l` | **Lowercase** | Automatically coerces assigned strings to lowercase. |
| `-u` | **Uppercase** | Automatically coerces assigned strings to uppercase. |
| `-x` | **Export** | Exports the variable to the environment for child processes. |
| `-p` | **Print** | Displays the internal definition and attributes of a variable for debugging. |

The following script demonstrates how `declare` attributes enforce data integrity for integers, read-only paths, and case-normalized environment tokens:

```bash
# Integer flag (-i) forces automatic arithmetic evaluation
declare -i WORKER_COUNT=2
WORKER_COUNT=WORKER_COUNT+3
echo "Total workers: $WORKER_COUNT"
# Output: Total workers: 5

# Read-only flag (-r) locks the variable against modification
declare -r APP_CONFIG="/etc/app/production.conf"
# APP_CONFIG="/tmp/override.conf" # This command would fail with: APP_CONFIG: readonly variable

# Case conversion flags (-l and -u)
declare -l ENVIRONMENT_TAG="STAGING-WEST"
echo "Normalized tag: $ENVIRONMENT_TAG"
# Output: Normalized tag: staging-west

# Debug inspection flag (-p) shows internal attribute definitions
declare -p ENVIRONMENT_TAG
# Output: declare -l ENVIRONMENT_TAG="staging-west"
```

#### Working with Indexed Arrays

Indexed arrays store ordered lists where elements are accessed using numeric zero-based indices (`0`, `1`, `2`, ...).

> [!WARNING]
> When appending elements to an array, the parentheses in `ARRAY+=("new_item")` are mandatory. If you omit the parentheses and write `ARRAY+="new_item"`, Bash treats the operation as string concatenation against index `0`, corrupting the first element rather than adding a new element.

The following script demonstrates declaring an indexed array, appending an element safely, reading individual elements, retrieving all elements, and calculating total length:

```bash
# 1. Declaration of an indexed array with initial values
declare -a SERVERS=("web-01" "web-02" "db-01")

# 2. Appending a new element using the +=() operator
SERVERS+=("cache-01")

# 3. Accessing a single element by its zero-based index
echo "First server: ${SERVERS[0]}"
# Output: First server: web-01

# 4. Accessing ALL elements simultaneously using the [@] symbol
echo "All servers: ${SERVERS[@]}"
# Output: All servers: web-01 web-02 db-01 cache-01

# 5. Retrieving the total element count using the '#' prefix
echo "Total managed nodes: ${#SERVERS[@]}"
# Output: Total managed nodes: 4
```

---

### 1.4 Dictionaries (Associative Arrays)

Associative arrays store key-value mappings, allowing you to associate arbitrary text keys with specific values (analogous to a dictionary in Python or a hash map in other languages). In DevOps engineering, associative arrays are widely used to map hostnames to IP addresses, service names to port numbers, or usernames to access tiers.

> [!IMPORTANT]
> You **must** declare associative arrays using `declare -A` before assigning values. If you attempt to assign a key-value pair without prior `-A` declaration, Bash evaluates the string key as an arithmetic expression, which results in syntax errors or corrupted numeric indexing.

Associative arrays use distinct syntax operators for keys and values:
- `${ARRAY["key"]}`: Returns the value mapped to `"key"`.
- `${ARRAY[@]}`: Returns all values stored in the array (order is not guaranteed).
- `${!ARRAY[@]}`: The exclamation mark (`!`) operator returns all **keys** in the array.
- `${#ARRAY[@]}`: Returns the total number of key-value pairs.
- `unset ARRAY["key"]`: Removes a specific key-value pair from the array.

The following script demonstrates declaring an associative array, assigning key-value mappings, querying keys and values, unsetting an entry, and iterating over the map:

```bash
# 1. Mandatory declaration of an associative array
declare -A HOST_IPS

# 2. Assigning key-value mappings
HOST_IPS["web-01"]="10.0.0.5"
HOST_IPS["db-01"]="10.0.0.10"
HOST_IPS["cache-01"]="10.0.0.15"

# 3. Accessing a specific value by key
echo "Database IP: ${HOST_IPS["db-01"]}"
# Output: Database IP: 10.0.0.10

# 4. Accessing all values (notice order is not guaranteed in hash lookups)
echo "All assigned IPs: ${HOST_IPS[@]}"

# 5. Accessing all keys using the '!' prefix
echo "All registered hostnames: ${!HOST_IPS[@]}"

# 6. Removing a key-value entry using unset
unset HOST_IPS["cache-01"]
echo "Remaining host count: ${#HOST_IPS[@]}"
# Output: Remaining host count: 2

# 7. Iterating over keys and values using a for loop
for HOST in "${!HOST_IPS[@]}"; do
    echo "Host: $HOST -> IP: ${HOST_IPS[$HOST]}"
done
```

---

## 2. Advanced Workflow Constructions

Production automation demands structured branching and robust looping mechanisms. Bash provides modern conditional constructs, pattern-matching routers, and resilient stream parsers.

---

### 2.1 Advanced `if` Statements

Modern Bash scripts rely on the double-bracket syntax `[[ ... ]]` for conditional evaluations. Compared to the legacy single-bracket `[ ... ]` command, double brackets offer three significant engineering advantages:
1. They prevent syntax errors caused by word splitting on unquoted or empty variables.
2. They support native logical operators (`&&` for AND, `||` for OR) directly inside the brackets.
3. They allow regex and pattern-matching operations without invoking external subshells.

The table below lists the standard conditional test operators used in infrastructure automation:

| Operator | Evaluates To True If: | Common DevOps Use Case |
| :--- | :--- | :--- |
| `-f "$PATH"` | Path exists and is a regular file. | Checking for config files or certificates. |
| `-d "$PATH"` | Path exists and is a directory. | Verifying mount points or log destinations. |
| `-r "$PATH"` | Path exists and current user has read permission. | Validating configuration readability before loading. |
| `-w "$PATH"` | Path exists and current user has write permission. | Validating backup directory write access. |
| `-x "$PATH"` | Path exists and is executable. | Ensuring custom daemon binaries can run. |
| `-s "$PATH"` | Path exists and size is greater than zero bytes. | Ensuring a generated report is not empty. |
| `-z "$VAR"`  | String length is zero (string is empty or unset). | Validating required CLI arguments. |
| `-n "$VAR"`  | String length is non-zero (string contains text). | Checking whether an environment variable was set. |

#### Part A: Multi-Condition Chains (`&&` and `||`)

In production environments, checking a single condition is rarely sufficient. For example, confirming a configuration file exists does not guarantee your process has read permission to open it.

The following script evaluates both file existence (`-f`) and read permissions (`-r`) simultaneously using the `&&` operator:

```bash
TARGET_FILE="/etc/hosts"

# -f checks for a regular file; -r checks read permissions
if [[ -f "$TARGET_FILE" && -r "$TARGET_FILE" ]]; then
    echo "File exists and is readable. Proceeding with configuration audit..."
elif [[ -f "$TARGET_FILE" ]]; then
    echo "File exists, but current user lacks read permissions. Elevated access required."
else
    echo "Error: Target configuration file does not exist."
fi
```

#### Part B: Inline Command Checking

In Bash, every command returns an integer Exit Status (Exit Code) ranging from `0` to `255`. By convention, **Exit Code 0 indicates success**, while any non-zero code indicates an error.

An `if` statement can evaluate command exit statuses directly. If the command succeeds (Exit Code 0), the `then` branch executes; otherwise, the `else` branch executes. In scripts, standard output (`stdout`) and standard error (`stderr`) are frequently redirected to `/dev/null` using `> /dev/null 2>&1` so only the exit code influences script execution.

The following script executes an ICMP network probe and branches based on whether the packet was successfully acknowledged:

```bash
# Silence command output so only the exit code is evaluated by the if condition
if ping -c 1 8.8.8.8 > /dev/null 2>&1; then
    echo "Gateway network check: UP (Exit Code 0)"
else
    echo "Gateway network check: DOWN (Non-zero exit code)"
fi
```

#### Part C: Nested `if` Statements

When a secondary decision depends entirely on the outcome of a primary decision, nest `if` blocks inside one another. Strict indentation is essential to preserve readability and prevent logical bugs in multi-tier verification workflows.

The following script demonstrates a nested validation check that verifies user role eligibility before inspecting multi-factor authentication (MFA) status:

```bash
USER_TYPE="admin"
MFA_ENABLED="false"

if [[ "$USER_TYPE" == "admin" ]]; then
    echo "Administrative access requested. Checking security policies..."
    
    # Nested condition: evaluated strictly when primary admin check passes
    if [[ "$MFA_ENABLED" == "true" ]]; then
        echo "MFA token validated. Administrative access granted."
    else
        echo "Security violation: Administrators must have MFA enabled. Access denied."
    fi
else
    echo "Standard user login verified. Access granted."
fi
```

---

### 2.2 CLI Routers (`case` Statements)

When authoring management utilities that accept command-line flags (such as `--start`, `--stop`, or `--status`), chaining multiple `if / elif / else` blocks becomes verbose and difficult to maintain. The `case` statement provides pattern-matching branching optimized for CLI routing.

#### `case` Syntax Structure
- `case "$VARIABLE" in`: Initiates evaluation of the target variable.
- `"pattern")`: Matches a specific string or pattern. Multiple matching aliases can be separated with a pipe (`"start" | "--start"`).
- `;;`: The mandatory branch terminator. It halts execution of the `case` construct and jumps to `esac` (analogous to `break` in C or Python).
- `*)`: The wildcard catch-all branch that matches any input not handled by prior patterns (used for error handling and usage syntax prompts).
- `esac`: Closes the `case` construct (`case` spelled backwards).

#### Parameter Expansion Default Values: `${VAR:-default}`

CLI scripts frequently accept optional secondary arguments. The parameter expansion syntax `${2:-5}` evaluates to the value of `$2` if it is set and non-empty; otherwise, it falls back to the default value `5`.

The following script implements a complete production system manager utility that routes tasks based on the primary argument and utilizes default parameter fallback for optional line counts:

```bash
#!/bin/bash
# sys-manager.sh - Production System Management Router

ACTION=$1
# Default to 5 log lines if the second parameter ($2) is omitted
LINES=${2:-5}

case "$ACTION" in
    "--status" | "-s")
        echo "Running system uptime and load average audit..."
        uptime
        ;;
    "--disk" | "-d")
        echo "Inspecting root filesystem capacity..."
        df -h /
        ;;
    "--logs" | "-l")
        echo "Retrieving the last $LINES system log entries..."
        sudo journalctl -n "$LINES" --no-pager
        ;;
    *)
        echo "Error: Unrecognized or missing option: '$ACTION'"
        echo "Usage: ./sys-manager.sh [--status | --disk | --logs [lines]]"
        exit 1
        ;;
esac
```

---

### 2.3 Advanced `for` Loops

The `for` loop is the primary mechanism for executing operations against collections of items, groups of files, or numeric sequences.

#### Part A: Iterating Over an Indexed Array

When iterating over an array, always wrap the array reference in double quotes: `"${ARRAY[@]}"`. If array elements contain spaces (such as `"web 01"`), omitting quotes causes word splitting to fragment single elements into multiple iterations.

The following script iterates through a defined server inventory and executes a provisioning message for each node:

```bash
declare -a SERVERS=("web-01" "web-02" "db-01")

# The double quotes around "${SERVERS[@]}" protect against whitespace word splitting
for SERVER in "${SERVERS[@]}"; do
    echo "Initiating automated deployment pipeline on: $SERVER"
done
```

#### Part B: Iterating Over Files (Globbing)

File globbing allows loops to process files matching a wildcard pattern (such as `*.conf`). 

> [!TIP]
> By default, if no files match a glob pattern in Bash, the interpreter passes the unexpanded literal string (such as `"*.conf"`) into the loop. Always include a defensive check (`if [[ ! -f "$FILE" ]]; then continue; fi`) to skip execution if no files actually match the pattern.

The following script iterates over all configuration files in the current working directory, skipping non-existent matches and backing up active files:

```bash
# Process all .conf files in the current working directory
for FILE in *.conf; do
    # Guard clause: skip if the glob did not match any real files
    if [[ ! -f "$FILE" ]]; then
        continue
    fi
    
    echo "Creating safety backup for configuration file: $FILE"
    # cp "$FILE" "${FILE}.bak"
done
```

#### Part C: C-Style Counter Loops

When you need to execute an operation a precise number of times without a predefined list (for example, generating sequential worker nodes or managing retry intervals), use C-style three-expression syntax: `for (( init; condition; step ))`.

The following script uses a C-style counter loop to create three sequentially numbered test identifiers:

```bash
# Three-expression counter: initialization; termination condition; step increment
for (( i=1; i<=3; i++ )); do
    echo "Provisioning staging test resource: QA-Worker-$i"
done
# Output:
# Provisioning staging test resource: QA-Worker-1
# Provisioning staging test resource: QA-Worker-2
# Provisioning staging test resource: QA-Worker-3
```

---

### 2.4 Reading Streams Line-by-Line (`while read -r`)

When parsing structured files (such as `/etc/passwd`, CSV reports, or log exports), iterating with standard `for` loops fails because spaces within lines trigger word splitting. The production standard for stream parsing is the `while read -r` construct.

The `-r` flag prevents backslashes from acting as escape characters, ensuring literal characters like `\n` or `\t` are preserved exactly as written in the source stream.

#### Part A: Standard File Redirection

The most reliable way to process a static file is by redirecting it into the `while read` loop using the `<` input redirector at the end of the loop block.

The following script generates a mock host list, iterates through each line, and skips blank lines using the `-z` string test:

```bash
# Generate sample inventory file with an empty line
echo -e "host-01\nhost-02\n\nhost-03" > servers.txt

# Read stream line by line using input redirection
while read -r LINE; do
    # Skip empty lines to prevent processing empty arguments
    if [[ -z "$LINE" ]]; then
        continue
    fi
    echo "Dispatching health check to target: $LINE"
done < servers.txt
```

#### Part B: Reading Command Output via Pipes

When streaming the live output of a command directly into a loop, you can pipe (`|`) the command into `while read -r`.

The following script pipes directory listings into a `while` loop to process each line dynamically:

```bash
# Stream output directly from a command pipe into while read
ls -1 | while read -r FILE_NAME; do
    echo "Discovered filesystem entry: $FILE_NAME"
done
```

#### Part C: The Subshell Trap & Process Substitution

Piping command output into a loop (`command | while read -r LINE`) introduces an architectural pitfall: **The Subshell Trap**.

In Bash, every command in a pipeline runs in a separate child process (subshell). Variables modified or counters incremented inside a piped `while` loop exist only inside that temporary child process. When the loop terminates, the subshell exits, and all modified variable state is discarded.

To avoid variable loss while processing dynamic command output, use **Process Substitution**: `< <(command)`. This syntax executes the command in a file descriptor and feeds it into the `while` loop using redirection (`<`). Because the loop executes in the primary shell context, all variable modifications persist after the loop concludes.

The following script demonstrates the subshell amnesia problem and how Process Substitution safely preserves counter tallies:

```bash
ERROR_COUNT=0

# Process Substitution: feeds command output directly without creating a loop subshell
while read -r LOG_LINE; do
    if [[ "$LOG_LINE" == *"ERROR"* ]]; then
        ((ERROR_COUNT++))
    fi
done < <(echo -e "INFO: Service started\nERROR: Connection refused\nERROR: Connection timeout")

# Because process substitution was used, ERROR_COUNT retains its updated value
echo "Total critical errors identified: $ERROR_COUNT"
# Output: Total critical errors identified: 2
```

---

## 3. Summary & Best Practices Matrix

When authoring automation scripts in production environments, enforce these architectural guidelines:

1. **Quote all variable expansions:** Always use `"$VAR"` and `"${ARRAY[@]}"` to prevent word splitting and globbing bugs.
2. **Declare associative arrays explicitly:** Always invoke `declare -A` before assigning key-value mappings.
3. **Use double brackets for conditions:** Prefer `[[ ... ]]` over `[ ... ]` for safer multi-condition checks and pattern comparisons.
4. **Use `-r` with `read`:** Always write `while read -r` to ensure backslashes are not stripped from input streams.
5. **Prefer Process Substitution over pipes:** When updating variables or tracking counts from command output, use `while read -r LINE; do ... done < <(command)` to avoid subshell variable loss.
