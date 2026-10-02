# Module 7: Advanced Shell Scripting & Automation

## Homework Assignment: Server Provisioning Router

**Scenario:**
You are writing a master automation script that handles server provisioning. The script needs to accept a command-line flag (`--web` or `--db`), read a configuration list of target IPs, and output a simulated provisioning log.

### Task 1: The Config File
1. Create a file named `targets.txt`.
2. Add the following IP addresses, including a blank line in the middle:
```text
10.0.0.5
10.0.0.6

10.0.0.10
```

### Task 2: The Script (`provision.sh`)
Write a bash script that does the following:

1. **Parameter Checking:** Use a `case` statement to check `$1` (the first argument).
   - If it is `--web`, set a variable `ROLE="Web Server"`.
   - If it is `--db`, set a variable `ROLE="Database Server"`.
   - If it is anything else, print a usage message: `Usage: ./provision.sh [--web | --db]` and `exit 1`.
2. **File Checking:** Use an `if` statement to check if `targets.txt` exists and is readable (`-f` and `-r`). If not, print an error and exit.
3. **Data Processing:**
   - Create an empty indexed array `declare -a SUCCESSFUL_PROVISIONS=()`.
   - Use a `while read -r` loop to process `targets.txt` line by line.
   - If a line is completely empty (`-z`), skip it using `continue`.
   - Inside the loop, `echo` that you are "Provisioning $ROLE at IP: $LINE".
   - Append the `$LINE` to the `SUCCESSFUL_PROVISIONS` array.
4. **Summary Report:** After the loop finishes, print out the total number of servers successfully provisioned by evaluating the length of the array (`${#SUCCESSFUL_PROVISIONS[@]}`).

### Expected Output
```bash
$ ./provision.sh --web
Provisioning Web Server at IP: 10.0.0.5
Provisioning Web Server at IP: 10.0.0.6
Provisioning Web Server at IP: 10.0.0.10
Total servers provisioned: 3
```
