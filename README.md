<div align="center">

# ⚡ CrossFire v4.0 - BlackBase (Release)

**The Universal Package Management Revolution**
*One Command. Every Platform. Total Control.*

</div>

---

💡 **What is CrossFire?**
CrossFire is a **universal package manager** that unifies your entire software ecosystem under a single, intelligent command-line interface. Built in Python, it **automates detection, installation, and management of packages** across multiple platforms and ecosystems.

Say goodbye to juggling pip, npm, apt, brew, winget, and dozens of others. With CrossFire, **one command is all you need**.

---

## 🚀 Key Features

* **Intelligent Manager Selection**
  Auto-detects your OS, architecture, and installed managers to pick the **optimal tool** for each task.

* **Unified Command Arsenal**
  Use the same commands to install, remove, and update packages, **regardless of the underlying manager**.

* **Comprehensive Coverage**
  Supports **pip, npm, apt, dnf, pacman, brew, winget, chocolatey, snap, flatpak**, and more.

* **Performance Engineered**
  Concurrent operations and smart caching speed up installations and updates.

* **Self-Sustaining Architecture**
  Single-file, **dependency-free deployment** with self-update capabilities.

---

## 🛠️ Quick Start

### Installation

Get started in **one line**:

```bash
# Download, install, and configure CrossFire
curl -o crossfire.py https://raw.githubusercontent.com/BCAS-Team/CrossFire/main/CrossFireL/crossfire.py \
  && chmod +x crossfire.py \
  && python crossfire.py --setup
```

The `--setup` command performs:

* **PATH Integration**: Adds `crossfire.py` to your user-local bin (`~/.local/bin`) and system PATH.
* **Shell Configuration**: Auto-detects `.bashrc`, `.zshrc`, or `.fish` and updates it.
* **Windows Integration**: Generates a `.bat` launcher for smooth Windows use.

---

## 🎯 Command Arsenal

### Core Operations

```bash
# Smart Installation (auto-detects best manager)
crossfire -i numpy

# Force a specific manager
crossfire -i numpy --manager pip

# Batch Installation from a file
crossfire --install-from requirements.txt

# Package Removal
crossfire -r package_name

# System-wide updates
crossfire --update-managers

# Health check
crossfire --health
```

### Advanced Operations

```bash
# Verbose debugging
crossfire -v -i package_name

# JSON output for scripts
crossfire --json --list-managers

# Silent mode
crossfire -q -i package_name

# Configuration override
crossfire -i package --manager pip --verbose --json
```

---

## 🏗️ Architecture

CrossFire is built on a **modular, self-sustaining architecture** designed for **speed, reliability, and maintainability**.

```
┌──────────────────────────────────────┐
│        🔥 CrossFire Core Engine      │
├──────────────────────────────────────┤
│ 🧠 Intelligence Layer                │
│ ├─ System & Manager Detection        │
│ ├─ Context-Aware Selection           │
│ └─ Performance Analytics             │
├──────────────────────────────────────┤
│ ⚡ Execution Engine                  │
│ ├─ Concurrent Operations             │
│ ├─ Smart Caching & Retry Logic       │
│ └─ Timeout Management                │
├──────────────────────────────────────┤
│ 📡 Package Manager Interface         │
│ ├─ Linux (apt, dnf, pacman...)       │
│ ├─ Windows (winget, chocolatey)      │
│ ├─ macOS (homebrew)                  │
│ ├─ Language-Specific (pip, npm)      │
│ └─ Universal (snap, flatpak)         │
└──────────────────────────────────────┘
```

---

## 🤝 Contributing

Join a **community of digital revolutionaries** shaping the future of package management.

* **Bug Reports**: Open an issue with steps to reproduce.
* **Feature Requests**: Propose new features with a clear use case.
* **Code Contributions**: Fork, branch, and submit a pull request.

**Want to add a new package manager?** Follow our [Extension Guide](https://bcas-team.github.io/Crossfire/).

---

## 📚 Documentation

Full command reference, configuration options, and technical guides are available in our **[official documentation](https://bcas-team.github.io/Crossfire/)**.

---

## 🔗 Connect with the Revolution

📧 **Email**: [bcas.public@gmail.com](mailto:bcas.public@gmail.com)

---

## 📄 License

This project is licensed under the **MIT License** — because freedom matters.

<div align="center">

Made with 💡, grit, and a hint of rebellion.

© 2025 BCAS Team – Redefining the Digital Frontier

</div>
