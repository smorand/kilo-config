Detect the programming language used in this project and run the appropriate compliance subagent.

1. Look for `go.mod` to detect Go, `pyproject.toml` / `setup.py` to detect Python, or `Cargo.toml` to detect Rust
2. If Go: use the Task tool with `golang-compliance-checker` subagent
3. If Python: use the Task tool with `python-skill-compliance` subagent
4. If Rust: use the Task tool with `rust-compliance-checker` subagent
5. If none is detected: inform the user that no supported language was found

Pass any arguments provided as context: $ARGUMENTS
