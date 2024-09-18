
# Coding Standards

This document outlines the coding standards for [Organization Name].

## General Guidelines

- Use **meaningful variable and function names**.
- Keep functions **small and focused** on a single task.
- Avoid deep nesting of code. Break down complex code into smaller functions.
- Ensure all code is **documented** with clear and concise comments.

## Python Standards

- Follow **PEP 8** guidelines for Python code.
- Use `black` for code formatting and ensure the code is well-structured.
  
```bash
black your_code.py
```

## Git Commit Messages

- **Write meaningful commit messages**. The message should clearly state what was changed and why.
- Use the format:
  ```
  type: brief description
  ```
  Example:
  ```
  feat: Add user authentication
  ```

## Testing

- All new features should be covered with unit tests.
- Run tests before pushing to the repository:
  ```bash
  pytest
  ```

