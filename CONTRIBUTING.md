# Contributing to ECS Fargate Web Application

First off, thank you for considering contributing to this project! 🎉

## Table of Contents
- [Code of Conduct](#code-of-conduct)
- [How Can I Contribute?](#how-can-i-contribute)
- [Development Setup](#development-setup)
- [Pull Request Process](#pull-request-process)
- [Style Guidelines](#style-guidelines)
- [Commit Message Guidelines](#commit-message-guidelines)

---

## Code of Conduct

This project adheres to a code of conduct. By participating, you are expected to uphold this code. Please be respectful and constructive in all interactions.

---

## How Can I Contribute?

### Reporting Bugs

Before creating bug reports, please check existing issues to avoid duplicates. When creating a bug report, include:

- **Description**: Clear description of the issue
- **Steps to Reproduce**: Detailed steps to reproduce the behavior
- **Expected Behavior**: What you expected to happen
- **Actual Behavior**: What actually happened
- **Environment**: AWS region, CLI version, Docker version
- **Screenshots**: If applicable
- **Logs**: Relevant CloudWatch logs or error messages

### Suggesting Enhancements

Enhancement suggestions are tracked as GitHub issues. When creating an enhancement suggestion, include:

- **Description**: Clear description of the enhancement
- **Use Case**: Why this enhancement would be useful
- **Proposed Solution**: Your ideas on how to implement it
- **Alternatives**: Alternative solutions you've considered

### Pull Requests

- Fill in the required template
- Follow the style guidelines
- Update documentation as needed
- Add tests if applicable
- Ensure CI/CD passes

---

## Development Setup

### Prerequisites
```bash
# Install required tools
aws --version    # AWS CLI 2.x+
docker --version # Docker 20.x+
git --version    # Git 2.x+
```

### Setup Steps

1. **Fork the repository**
   ```bash
   # Click 'Fork' on GitHub, then clone your fork
  git clone https://github.com/CloudFolksPublic/php-s3-app.git
   cd ecs-fargate-project
   ```

2. **Add upstream remote**
   ```bash
   git remote add upstream https://github.com/ORIGINAL_OWNER/ecs-fargate-project.git
   ```

3. **Create a branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

4. **Set up environment**
   ```bash
   cp .env.example .env
   # Edit .env with your AWS credentials
   ```

5. **Test your changes**
   ```bash
   # Deploy to a test environment
   ./scripts/deploy-test.sh
   
   # Verify functionality
   ./scripts/verify.sh
   ```

---

## Pull Request Process

1. **Update your fork**
   ```bash
   git fetch upstream
   git rebase upstream/main
   ```

2. **Make your changes**
   - Write clear, concise code
   - Follow existing patterns
   - Add comments for complex logic
   - Update documentation

3. **Test thoroughly**
   - Test in your AWS account
   - Verify all functionality works
   - Check for any breaking changes
   - Test cleanup/teardown process

4. **Commit your changes**
   ```bash
   git add .
   git commit -m "feat: add new feature description"
   ```

5. **Push to your fork**
   ```bash
   git push origin feature/your-feature-name
   ```

6. **Create Pull Request**
   - Go to GitHub and create a PR
   - Fill in the PR template
   - Link related issues
   - Request review

7. **Address feedback**
   - Respond to review comments
   - Make requested changes
   - Push updates to your branch

---

## Style Guidelines

### Code Style

#### Bash Scripts
```bash
#!/bin/bash
# Use descriptive variable names
VARIABLE_NAME="value"

# Add comments for complex operations
# This function creates a VPC with the specified CIDR block
create_vpc() {
    local cidr_block=$1
    # Function implementation
}

# Use error handling
if ! aws ecs describe-clusters --cluster my-cluster; then
    echo "Error: Cluster not found"
    exit 1
fi
```

#### JSON Files
```json
{
  "family": "task-name",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "container-name",
      "image": "image-uri"
    }
  ]
}
```

### Documentation Style

- Use clear, concise language
- Include code examples
- Add screenshots when helpful
- Keep formatting consistent
- Use markdown features appropriately

#### Headers
```markdown
# Main Title (H1)
## Section (H2)
### Subsection (H3)
```

#### Code Blocks
````markdown
```bash
# Always specify language
aws ecs list-clusters
```
````

#### Lists
```markdown
- Use bullet points for unordered lists
1. Use numbers for ordered lists
```

---

## Commit Message Guidelines

Follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

### Format
```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types
- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code style changes (formatting, etc.)
- **refactor**: Code refactoring
- **test**: Adding or updating tests
- **chore**: Maintenance tasks

### Examples

```
feat(ecs): add support for Fargate Spot

Added configuration for Fargate Spot capacity provider to reduce costs
by up to 70% for non-critical workloads.

Closes #123
```

```
fix(alb): correct health check configuration

Health check timeout was too short, causing false negatives.
Increased timeout from 5s to 10s.

Fixes #456
```

```
docs(readme): update deployment instructions

Added more detailed steps for network configuration and
troubleshooting common issues.
```

---

## Testing Guidelines

### Before Submitting PR

1. **Test Deployment**
   ```bash
   # Deploy to test account
   ./scripts/deploy-test.sh
   
   # Verify all resources created
   ./scripts/verify.sh
   ```

2. **Test Cleanup**
   ```bash
   # Ensure cleanup works properly
   ./scripts/cleanup.sh
   
   # Verify all resources deleted
   aws ecs list-clusters
   ```

3. **Test Documentation**
   - Follow your own instructions
   - Verify all commands work
   - Check for typos or errors

### Test Checklist
- [ ] Deployment script runs without errors
- [ ] All resources created successfully
- [ ] Application accessible via ALB
- [ ] Auto scaling works as expected
- [ ] Cleanup removes all resources
- [ ] Documentation is clear and accurate
- [ ] No hardcoded credentials or sensitive data

---

## Documentation Guidelines

### What to Document

- New features or functionality
- Configuration changes
- Architecture modifications
- Breaking changes
- Migration guides
- Troubleshooting tips

### Documentation Files

- **README.md**: Overview and quick start
- **DEPLOYMENT_GUIDE.md**: Detailed deployment steps
- **ARCHITECTURE.md**: Technical architecture details
- **TROUBLESHOOTING.md**: Common issues and solutions

---

## Review Process

### What We Look For

1. **Functionality**
   - Does it work as intended?
   - Are there any bugs?
   - Does it handle errors gracefully?

2. **Code Quality**
   - Is the code clean and readable?
   - Are variables named appropriately?
   - Is there adequate error handling?

3. **Documentation**
   - Is the documentation clear?
   - Are examples provided?
   - Is it easy to follow?

4. **Testing**
   - Has it been tested thoroughly?
   - Are edge cases considered?
   - Does cleanup work properly?

### Timeline

- Initial review: Within 2-3 days
- Follow-up reviews: 1-2 days
- Merge: After approval and passing tests

---

## Getting Help

If you need help or have questions:

1. **Check Documentation**: Review existing docs first
2. **Search Issues**: Look for similar questions/issues
3. **Ask in Discussions**: Use GitHub Discussions for questions
4. **Open an Issue**: Create an issue for bugs or feature requests

---

## Recognition

Contributors will be recognized in:
- README.md contributors section
- GitHub contributors page
- Release notes (for significant contributions)

---

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

Thank you for contributing! 🙌
