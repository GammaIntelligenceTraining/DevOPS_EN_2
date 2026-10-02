# Module 7: Advanced Shell Scripting & Automation

## 1. Bash Datatypes & Arithmetic

| Concept | Syntax Example | Description |
| :--- | :--- | :--- |
| **Strings** | `NAME="server-01"` | By default, all variables in Bash are strings. |
| **Concatenation** | `FULL="${NAME}.local"` | Safely combining strings using brackets. |
| **Integer Math** | `FREE=$(( 10 + 5 ))` | The `$(( ))` syntax forces mathematical evaluation (`+ - * /`). |
| **Modulo Math** | `REM=$(( 10 % 3 ))` | Returns the remainder of a division (useful for odd/even checks). |
| **Indexed Arrays** | `declare -a LIST=("A" "B")` | Creates an ordered list of items (zero-indexed). |
| **Array Append** | `LIST+=("C")` | Adds a new element to the end of the array. |
| **Array Access** | `echo "${LIST[0]}"` | Retrieves the first element of the array. |
| **Array Access All** | `echo "${LIST[@]}"` | Retrieves ALL elements of the array. |
| **Array Length** | `echo "${#LIST[@]}"` | Retrieves the total number of items in the array. |
| **Dictionaries** | `declare -A MAP` | Creates an associative array (Key-Value pairs). |
| **Dictionary Add** | `MAP["web"]="10.0.0.1"` | Assigns a value to a specific key. |
| **Dictionary Keys** | `echo "${!MAP[@]}"` | The `!` prefix extracts ALL Keys from the dictionary. |
| **Dictionary Values** | `echo "${MAP[@]}"` | Extracts ALL Values from the dictionary. |
| **Delete Variable** | `unset MAP["web"]` | Removes a key-value pair from an array (or a standard variable). |

---

## 2. Advanced Workflow Constructions

| Construction | Syntax Example | When to use it in DevOps |
| :--- | :--- | :--- |
| **Multi-Condition `if`** | `if [[ -f "$F" && -r "$F" ]]; then` | Checking if multiple prerequisites are met before acting. |
| **Inline Cmd Check** | `if ping -c 1 "$IP" > /dev/null; then` | Checking if a command succeeded (Exit Code 0). |
| **Nested `if`** | `if [[ "$X" ]]; then if [[ "$Y" ]]; then` | Executing sub-decisions (e.g., verifying admin, then checking MFA). |
| **CLI Router (`case`)** | `case "$1" in "start") ... ;; esac` | Building command-line tools that accept flags like `--start`, `--stop`, or `--status`. |
| **Array Loop (`for`)** | `for IP in "${LIST[@]}"; do` | Executing the same command against a known list of variables. |
| **Globbing Loop** | `for FILE in *.log; do` | Iterating over files in a directory that match a pattern. |
| **C-Style Loop** | `for (( i=1; i<=5; i++ )); do` | Running a command exactly N times (like creating sequential users). |
| **Standard Stream**| `while read -r LINE; do ... < file` | Reading a file line-by-line safely. |
| **Piped Stream**| `ls -1 \| while read -r F; do` | Reading the output of a command (Warning: runs in subshell!). |
| **Proc Substitution**| `while read -r L; do ... < <(cmd)` | The safest way to read command output without losing variable data to a subshell. |
| **Skip Iteration** | `if [[ -z "$L" ]]; then continue; fi` | Skipping empty lines or irrelevant data while inside a `for` or `while` loop. |
