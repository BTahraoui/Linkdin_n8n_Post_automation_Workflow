# Contributing

> How to contribute to this project and help make it better for everyone.

We love your input! We want to make contributing to this project as easy and transparent as possible, whether it's:
- Reporting a bug
- Discussing the current state of the code
- Submitting a fix
- Proposing new features
- Becoming a maintainer

## 📝 How to Contribute

### 1. Fork the repo
Create your own fork of the repository.

### 2. Clone your fork
```bash
git clone https://github.com/<your-username>/Linkdin_n8n_Post_automation_Workflow.git
cd Linkdin_n8n_Post_automation_Workflow
```

### 3. Create a branch
```bash
git checkout -b feature/amazing-feature
```
or
```bash
git checkout -b fix/issue-description
```

### 4. Make your changes
- Edit the workflow (`workflow.json` or `workflow.jason`)
- Update documentation as needed
- Add examples or test cases

### 5. Test your changes
- Import the workflow into your n8n instance
- Run it manually to verify it works
- Check that credentials are properly configured (use the sanitized version if sharing)

### 6. Commit and push
```bash
git add .
git commit -m "Add some amazing feature"
git push origin feature/amazing-feature
```

### 7. Open a Pull Request
- Go to your fork on GitHub
- Click "Compare & pull request"
- Fill in the PR template
- Describe what your change does and why it's needed

## 🐛 Reporting Bugs

Before submitting a bug report, please check if it's already been reported by searching existing Issues.

When you submit a bug report, include:
- **Version**: Which version of the workflow you're using
- **Environment**: Self-hosted or n8n.cloud? n8n version?
- **Steps to reproduce**: Clear, step-by-step instructions
- **Expected behavior**: What you expected to happen
- **Actual behavior**: What actually happened (include screenshots if helpful)
- **Error messages**: Exact text from n8n execution logs
- **Workflow export**: If possible, attach a sanitized version of your workflow

## 💡 Suggesting Features or Improvements

Feature requests are welcome! Please open an issue and include:
- **Clear description**: What you'd like to see and why
- **Use case**: How you would use this feature
- **Possible drawbacks**: Any potential downsides you can think of
- **Mockups or examples**: If applicable

## 📚 Documentation Improvements

We love documentation improvements! If you find typos, unclear sections, or missing information:
- Fix them directly in the Markdown files
- Or open an issue suggesting the improvement

## 🛠️ Code Style

While this project is primarily JSON (n8n workflow) and Markdown documentation:
- Keep JSON formatted with 2-space indentation
- Keep Markdown lines under 100 characters where reasonable
- Use consistent terminology (refer to nodes by their exact names)
- Keep the workflow readable — add comments via sticky notes if needed

## 🔐 Security

If you discover a security issue, please DO NOT open a public issue.
Instead, follow the guidelines in [SECURITY.md](docs/SECURITY.md).

## 🙏 Recognition

Contributors will be recognized in:
- The README.md
- The CREDITS.md file
- GitHub's contributor graph

## 📜 License

By contributing, you agree that your contributions will be licensed under the MIT License (see [LICENSE](LICENSE)).

---

*Made with ❤ for the Linkedin builder community.*