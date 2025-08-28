# CrossFire Universal Package Manager - Complete Technical Documentation

**Version:** CrossFire 1.0 - BlackBase  
**File:** crossfire.py  
**Python:** 3.x required  
**Dependencies:** requests (mandatory), distro (optional)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Global Configuration & Constants](#2-global-configuration--constants)
3. [OS & Platform Detection](#3-os--platform-detection)
4. [Logging & Console System](#4-logging--console-system)
5. [Database System](#5-database-system)
6. [Progress Tracking System](#6-progress-tracking-system)
7. [Network Testing & Speed Tools](#7-network-testing--speed-tools)
8. [Search Engine Implementation](#8-search-engine-implementation)
9. [Command Execution Framework](#9-command-execution-framework)
10. [Package Manager Detection & Commands](#10-package-manager-detection--commands)
11. [Installation & Removal Orchestration](#11-installation--removal-orchestration)
12. [System Maintenance & Cleanup](#12-system-maintenance--cleanup)
13. [Self-Update System](#13-self-update-system)
14. [System Information & Health Checks](#14-system-information--health-checks)
15. [Setup & Launcher Installation](#15-setup--launcher-installation)
16. [Package Management Features](#16-package-management-features)
17. [Command Line Interface](#17-command-line-interface)
18. [Main Entry Point & Routing](#18-main-entry-point--routing)
19. [Error Handling & Edge Cases](#19-error-handling--edge-cases)
20. [Security Considerations](#20-security-considerations)
21. [Known Bugs & Limitations](#21-known-bugs--limitations)
22. [Extension Guide](#22-extension-guide)
23. [Complete Command Reference](#23-complete-command-reference)

---

## 1. Architecture Overview

CrossFire is a single-file, zero-dependency-install universal package manager that orchestrates multiple system package managers (pip, npm, brew, apt, etc.) through a unified interface.

### Core Design Principles:
- **Single File Distribution**: Self-contained executable Python script
- **Zero External Config**: All configuration stored in `~/.crossfire/`
- **Multi-Manager Support**: Handles 13+ package managers across platforms
- **Real Search**: Actual API queries to PyPI, NPM, Homebrew
- **Package Tracking**: SQLite database tracks installations
- **Self-Updating**: Can update itself via HTTPS with verification

### File Structure:
```
crossfire.py
├── Imports & Constants (lines 1-50)
├── OS Detection (lines 51-80)
├── Logging System (lines 81-120)
├── Database Layer (lines 121-200)
├── Progress/UI Components (lines 201-280)
├── Network Tools (lines 281-400)
├── Search Engine (lines 401-650)
├── Command Execution (lines 651-720)
├── Manager Integrations (lines 721-950)
├── Install/Remove Logic (lines 951-1150)
├── System Tools (lines 1151-1400)
├── CLI Parser (lines 1401-1600)
├── Main Router (lines 1601-1800)
└── Entry Point (lines 1801-1820)
```

---

## 2. Global Configuration & Constants

### Version & URLs
```python
__version__ = "CrossFire 1.0 - BlackBase"
DEFAULT_UPDATE_URL = "https://raw.githubusercontent.com/crossfire-pm/crossfire-launcher/main/crossfire.py"
```

### Directory Structure
```python
CROSSFIRE_DIR = Path.home() / ".crossfire"        # ~/.crossfire/
CROSSFIRE_DB = CROSSFIRE_DIR / "packages.db"      # SQLite database
CROSSFIRE_CACHE = CROSSFIRE_DIR / "cache"         # Download cache
```

**Initialization:**
- Directories created with `exist_ok=True` on import
- No permission checks or error handling for read-only filesystems

---

## 3. OS & Platform Detection

### Core Detection Logic
```python
OS_NAME = platform.system()          # "Windows", "Darwin", "Linux"
ARCH = platform.architecture()[0]    # "64bit", "32bit"

# Distribution Detection (with fallback)
try:
    import distro
    DISTRO_NAME = distro.id() or "linux"
    DISTRO_VERSION = distro.version() or ""
except Exception:
    # Fallback logic for each OS
    if OS_NAME == "Darwin":
        DISTRO_NAME = "macOS"
        DISTRO_VERSION = platform.mac_ver()[0]
    elif OS_NAME == "Windows":
        DISTRO_NAME = "Windows"  
        DISTRO_VERSION = platform.version()
    else:
        DISTRO_NAME = OS_NAME.lower()
        DISTRO_VERSION = ""
```

**Usage Throughout Code:**
- Determines available package managers
- Influences installation command generation
- Affects setup/PATH modification behavior

---

## 4. Logging & Console System

### Colors Class
```python
class Colors:
    INFO = "\033[94m"      # Blue
    SUCCESS = "\033[92m"   # Green  
    WARNING = "\033[93m"   # Yellow
    ERROR = "\033[91m"     # Red
    MUTED = "\033[90m"     # Gray
    BOLD = "\033[1m"       # Bold
    CYAN = "\033[96m"      # Cyan
    RESET = "\033[0m"      # Reset
```

### Logger Implementation
```python
class Logger:
    def __init__(self):
        self.quiet = False      # Suppress INFO/WARNING/SUCCESS
        self.verbose = False    # Show debug info & tracebacks  
        self.json_mode = False  # Suppress all cprint output

    def cprint(self, text, color="INFO"):
        # JSON mode: suppress all output
        if self.json_mode:
            return
        
        # Quiet mode: only show ERROR
        if self.quiet and color in ["INFO", "WARNING", "SUCCESS"]:
            return
            
        # TTY detection for colors
        if not sys.stdout.isatty():
            sys.stdout.write(f"{text}\n")
            return
            
        color_code = getattr(Colors, color.upper(), Colors.INFO)
        print(f"{color_code}{text}{Colors.RESET}")
```

**Global Usage:**
```python
LOG = Logger()  # Global instance
cprint = LOG.cprint  # Convenience reference used throughout
```

**Behavior Notes:**
- JSON mode completely suppresses console output for machine parsing
- Quiet mode still allows ERROR messages through
- Colors automatically disabled for non-TTY output (pipes, redirects)

---

## 5. Database System

### Schema Design
```sql
CREATE TABLE IF NOT EXISTS installed_packages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    version TEXT,
    manager TEXT NOT NULL,
    install_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    install_command TEXT,
    UNIQUE(name, manager)
);
```

### PackageDB Implementation
```python
class PackageDB:
    def __init__(self, db_path: Path = CROSSFIRE_DB):
        self.db_path = db_path
        self._init_db()  # Creates table if not exists
    
    def add_package(self, name: str, version: str, manager: str, command: str = ""):
        """Records successful installation with INSERT OR REPLACE"""
        conn = sqlite3.connect(self.db_path)
        try:
            conn.execute('''
                INSERT OR REPLACE INTO installed_packages 
                (name, version, manager, install_command) 
                VALUES (?, ?, ?, ?)
            ''', (name, version or "unknown", manager, command))
            conn.commit()
        finally:
            conn.close()
    
    def remove_package(self, name: str, manager: str = None):
        """Removes package record(s)"""
        conn = sqlite3.connect(self.db_path)
        try:
            if manager:
                conn.execute('DELETE FROM installed_packages WHERE name = ? AND manager = ?', 
                           (name, manager))
            else:
                conn.execute('DELETE FROM installed_packages WHERE name = ?', (name,))
            conn.commit()
        finally:
            conn.close()
    
    def get_installed_packages(self, manager: str = None) -> List[Dict]:
        """Returns list of installed packages, optionally filtered by manager"""
        conn = sqlite3.connect(self.db_path)
        try:
            if manager:
                cursor = conn.execute('''
                    SELECT name, version, manager, install_date 
                    FROM installed_packages 
                    WHERE manager = ? 
                    ORDER BY install_date DESC
                ''', (manager,))
            else:
                cursor = conn.execute('''
                    SELECT name, version, manager, install_date 
                    FROM installed_packages 
                    ORDER BY install_date DESC
                ''')
            
            columns = [desc[0] for desc in cursor.description]
            return [dict(zip(columns, row)) for row in cursor.fetchall()]
        finally:
            conn.close()
    
    def is_installed(self, name: str, manager: str = None) -> bool:
        """Checks if package is recorded as installed"""
        conn = sqlite3.connect(self.db_path)
        try:
            if manager:
                cursor = conn.execute(
                    'SELECT COUNT(*) FROM installed_packages WHERE name = ? AND manager = ?',
                    (name, manager)
                )
            else:
                cursor = conn.execute(
                    'SELECT COUNT(*) FROM installed_packages WHERE name = ?',
                    (name,)
                )
            return cursor.fetchone()[0] > 0
        finally:
            conn.close()
```

**Global Instance:**
```python
package_db = PackageDB()  # Used throughout the application
```

**Key Behaviors:**
- `INSERT OR REPLACE` updates existing entries (by name+manager unique constraint)
- `remove_package()` without manager removes ALL entries for that package name
- Database operations are not transactional across multiple packages
- No database schema versioning or migration system

---

## 6. Progress Tracking System

### ProgressBar Implementation
```python
class ProgressBar:
    def __init__(self, total, description, unit):
        self.total = total
        self.description = description  
        self.unit = unit               # "B", "packages", "hosts", etc.
        self.current = 0
        self.start_time = time.time()
        self.lock = threading.Lock()   # Thread-safe updates
        self.bar_length = 50           # Character width of progress bar
        self.terminal_width = shutil.get_terminal_size((80, 20)).columns

    def update(self, step=1):
        """Thread-safe progress increment"""
        with self.lock:
            self.current = min(self.current + step, self.total)
            self._draw_bar()

    def _draw_bar(self):
        """Renders progress bar with ETA and speed calculations"""
        if LOG.json_mode or not sys.stdout.isatty():
            return
        
        progress = self.current / self.total if self.total > 0 else 0
        percent = progress * 100
        filled_length = int(self.bar_length * progress)
        bar = '█' * filled_length + '-' * (self.bar_length - filled_length)
        
        elapsed = time.time() - self.start_time
        eta_str = "N/A"
        speed_str = ""
        
        if progress > 0 and elapsed > 0:
            remaining = (elapsed / progress) - elapsed if progress < 1 else 0
            if remaining > 3600:
                eta_str = f"{remaining/3600:.1f}h"
            elif remaining > 60:
                eta_str = f"{remaining/60:.1f}m"
            else:
                eta_str = f"{remaining:.1f}s"
            
            # Speed calculation for bytes
            if self.unit == "B" and elapsed > 0:
                speed = self.current / elapsed
                if speed > 1024**2:
                    speed_str = f" @ {speed/1024**2:.1f} MB/s"
                elif speed > 1024:
                    speed_str = f" @ {speed/1024:.1f} KB/s"
                else:
                    speed_str = f" @ {speed:.0f} B/s"
        
        full_msg = f"{self.description}: |{bar}| {percent:.1f}% ({self.current}/{self.total} {self.unit}){speed_str} - ETA: {eta_str}"
        
        # Truncate if too long for terminal
        if len(full_msg) > self.terminal_width:
            full_msg = full_msg[:self.terminal_width - 4] + "..."
        
        sys.stdout.write(f"\r{full_msg}")
        sys.stdout.flush()

    def finish(self):
        """Completes progress bar and moves to new line"""
        if not LOG.json_mode and sys.stdout.isatty():
            sys.stdout.write("\n")
            sys.stdout.flush()
```

**Usage Patterns:**
- Network downloads (bytes with speed calculation)
- Multi-step operations (package installations, manager updates)
- Search operations across multiple repositories

---

## 7. Network Testing & Speed Tools

### SpeedTest Class

#### Download Speed Testing
```python
@staticmethod
def test_download_speed(url: Optional[str] = None, duration: int = 10) -> Dict[str, Any]:
    """Tests internet download speed using public speed test files"""
    
    # Default test URLs (fallback chain)
    test_urls = [
        "http://speedtest.tele2.net/10MB.zip",
        "https://proof.ovh.net/files/10Mb.dat", 
        "http://ipv4.download.thinkbroadband.com/10MB.zip"
    ]
    
    test_url = url or test_urls[0]
    downloaded_bytes = 0
    start_time = time.time()
    
    try:
        request = urllib.request.Request(test_url)
        with urllib.request.urlopen(request, timeout=30) as response:
            total_size = int(response.info().get("Content-Length", 10*1024*1024))
            tracker = ProgressBar(min(total_size, 50*1024*1024), "Speed Test", "B")
            
            while time.time() - start_time < duration:
                chunk = response.read(32768)  # 32KB chunks
                if not chunk:
                    break
                downloaded_bytes += len(chunk)
                tracker.update(len(chunk))
                
            tracker.finish()
        
        elapsed_time = time.time() - start_time
        download_rate_mbps = (downloaded_bytes * 8) / elapsed_time / 1000000 if elapsed_time > 0 else 0
        
        return {
            "ok": True,
            "download_mbps": round(download_rate_mbps, 2),
            "downloaded_mb": round(downloaded_bytes / 1024 / 1024, 2),
            "elapsed_seconds": round(elapsed_time, 2),
        }
        
    except Exception as e:
        return {"ok": False, "error": str(e)}
```

#### Ping Testing
```python
@staticmethod
def ping_test() -> Dict[str, Any]:
    """Tests network latency to multiple hosts"""
    
    hosts = ["google.com", "github.com", "cloudflare.com", "8.8.8.8"]
    results = {}
    progress = ProgressBar(len(hosts), "Ping test", "hosts")
    
    for host in hosts:
        try:
            # Platform-specific ping commands
            if OS_NAME == "Windows":
                command = ["ping", "-n", "1", "-w", "5000", host]
            else:
                command = ["ping", "-c", "1", "-W", "5", host]
                
            process = subprocess.run(command, capture_output=True, text=True, timeout=10)
            output = process.stdout
            
            # Parse latency from output (platform-specific regex)
            if OS_NAME == "Windows":
                latency_match = re.search(r"time[<=](\d+)ms", output)
            else:
                latency_match = re.search(r"time=(\d+\.?\d*)\s*ms", output)
            
            if latency_match:
                latency = float(latency_match.group(1))
                results[host] = {"ok": True, "latency_ms": latency}
            else:
                results[host] = {"ok": False, "msg": "Could not parse ping output"}
                
        except subprocess.TimeoutExpired:
            results[host] = {"ok": False, "msg": "Timed out"}
        except Exception as e:
            results[host] = {"ok": False, "msg": str(e)}
        
        progress.update()
        
    progress.finish()
    return results
```

**Limitations:**
- Ping regex parsing varies by OS/locale and may fail
- No packet loss measurement
- Speed test limited to HTTP downloads (no upload testing)
- Firewall/proxy configurations may interfere

---

## 8. Search Engine Implementation

### SearchResult Data Model
```python
@dataclass
class SearchResult:
    name: str
    description: str  
    version: str
    manager: str                    # "pip", "npm", "brew", etc.
    homepage: Optional[str] = None
    relevance_score: float = 0.0    # 0-100+ scoring for ranking
    
    def to_dict(self):
        return asdict(self)  # For JSON serialization
```

### RealSearchEngine Architecture
```python
class RealSearchEngine:
    def __init__(self):
        self.cache_timeout = 3600  # 1 hour for cached data
        self.session = requests.Session()
        self.session.timeout = 30
    
    def search(self, query: str, manager: Optional[str] = None, limit: int = 20) -> List[SearchResult]:
        """Orchestrates concurrent search across multiple package managers"""
        
        all_results = []
        installed = _detect_installed_managers()  # Only search available managers
        
        # Target manager selection
        if manager:
            target_managers = [manager.lower()] if installed.get(manager.lower()) else []
        else:
            target_managers = [m for m, ok in installed.items() if ok]
        
        if not target_managers:
            return []
        
        # Manager-specific search function mapping
        manager_funcs = {
            "pip": self._search_pypi,
            "npm": self._search_npm, 
            "brew": self._search_brew,
            "apt": self._search_apt,
            "dnf": self._search_dnf,
            "yum": self._search_yum,
            "pacman": self._search_pacman,
            "zypper": self._search_zypper,
            "apk": self._search_apk,
            "choco": self._search_choco,
            "winget": self._search_winget,
            "snap": self._search_snap,
            "flatpak": self._search_flatpak,
        }
        
        # Concurrent execution with timeout
        with _fut.ThreadPoolExecutor(max_workers=5) as executor:
            future_to_manager = {}
            for mgr in target_managers:
                func = manager_funcs.get(mgr)
                if func:
                    future_to_manager[executor.submit(func, query)] = mgr
            
            progress = ProgressBar(len(future_to_manager), "Searching repositories", "repos")
            for future in _fut.as_completed(future_to_manager, timeout=120):
                mgr = future_to_manager[future]
                try:
                    results = future.result() or []
                    all_results.extend(results)
                except Exception as e:
                    cprint(f"{mgr.upper()}: Search failed - {e}", "WARNING")
                finally:
                    progress.update()
            progress.finish()
        
        # Sort by relevance and limit results
        all_results.sort(key=lambda x: x.relevance_score, reverse=True)
        return all_results[:limit]
```

### Repository-Specific Search Implementations

#### PyPI Search
```python
def _search_pypi(self, query: str) -> List[SearchResult]:
    """PyPI direct package lookup (no full-text search available)"""
    try:
        url = f"https://pypi.org/pypi/{query}/json"
        r = self.session.get(url, timeout=10)
        if r.status_code == 200:
            return [self._parse_pypi_info(r.json())]
    except:
        pass
    return []

def _parse_pypi_info(self, data: Dict[str, Any]) -> SearchResult:
    """Converts PyPI API response to SearchResult"""
    info = data.get("info", {})
    return SearchResult(
        name=info.get("name", ""),
        description=info.get("summary", "")[:200],  # Truncate long descriptions
        version=info.get("version", "unknown"),
        manager="pip",
        homepage=info.get("home_page") or info.get("project_url"),
        relevance_score=50  # Fixed score for direct matches
    )
```

#### NPM Search  
```python
def _search_npm(self, query: str) -> List[SearchResult]:
    """Uses NPM registry search API with quality scoring"""
    try:
        url = "https://registry.npmjs.org/-/v1/search"
        r = self.session.get(url, params={"text": query, "size": 10}, timeout=15)
        if r.status_code != 200:
            return []
            
        data = r.json()
        results = []
        for obj in data.get("objects", []):
            pkg = obj.get("package", {})
            score = obj.get("score", {}).get("final", 0) * 100  # Convert to 0-100
            results.append(SearchResult(
                name=pkg.get("name", ""),
                description=pkg.get("description", "")[:200],
                version=pkg.get("version", "unknown"),
                manager="npm",
                homepage=pkg.get("homepage") or pkg.get("repository", {}).get("url"),
                relevance_score=score
            ))
        return results
    except:
        return []
```

#### Homebrew Search
```python
def _search_brew(self, query: str) -> List[SearchResult]:
    """Downloads Homebrew formula list and performs local search"""
    try:
        url = "https://formulae.brew.sh/api/formula.json"
        cache_file = CROSSFIRE_CACHE / "brew_formulae.json"
        
        # Use cached data if recent
        if cache_file.exists() and (time.time() - cache_file.stat().st_mtime < self.cache_timeout):
            formulae = json.load(open(cache_file))
        else:
            r = self.session.get(url, timeout=20)
            if r.status_code != 200:
                return []
            formulae = r.json()
            json.dump(formulae, open(cache_file, "w"))  # Cache for future searches
        
        results = []
        for f in formulae:
            name, desc = f.get("name", ""), f.get("desc", "")
            score = 0
            
            # Relevance scoring heuristics
            if query.lower() in name.lower(): 
                score += 50
            if query.lower() in desc.lower(): 
                score += 30
                
            if score > 0:
                results.append(SearchResult(
                    name=name, 
                    description=desc[:200],
                    version=f.get("versions", {}).get("stable", "unknown"),
                    manager="brew", 
                    homepage=f.get("homepage"), 
                    relevance_score=score
                ))
        
        return sorted(results, key=lambda x: x.relevance_score, reverse=True)[:10]
    except:
        return []
```

#### CLI-Based Searches (Linux Package Managers)
```python
def _cli_search(self, cmd: List[str], manager: str) -> List[SearchResult]:
    """Generic helper for CLI-based package manager searches"""
    res = run_command(cmd, timeout=30)
    results = []
    if res.ok:
        for line in res.out.splitlines():
            parts = line.strip().split(None, 1)  # Split on whitespace, max 2 parts
            if len(parts) >= 1:
                name = parts[0]
                desc = parts[1] if len(parts) > 1 else ""
                results.append(SearchResult(
                    name=name, 
                    description=desc[:200],
                    version="unknown", 
                    manager=manager, 
                    relevance_score=5  # Low score for CLI searches
                ))
    return results[:10]  # Limit results

# Specific implementations
def _search_apt(self, query: str): return self._cli_search(["apt-cache", "search", query], "apt")
def _search_dnf(self, query: str): return self._cli_search(["dnf", "search", query], "dnf")  
def _search_yum(self, query: str): return self._cli_search(["yum", "search", query], "yum")
def _search_pacman(self, query: str): return self._cli_search(["pacman", "-Ss", query], "pacman")
def _search_zypper(self, query: str): return self._cli_search(["zypper", "search", query], "zypper")
def _search_apk(self, query: str): return self._cli_search(["apk", "search", query], "apk")
def _search_choco(self, query: str): return self._cli_search(["choco", "search", query], "choco")
def _search_winget(self, query: str): return self._cli_search(["winget", "search", query], "winget")
def _search_snap(self, query: str): return self._cli_search(["snap", "find", query], "snap")
def _search_flatpak(self, query: str): return self._cli_search(["flatpak", "search", query], "flatpak")
```

**Global Instance:**
```python
search_engine = RealSearchEngine()  # Used by CLI search command
```

**Key Limitations:**
- PyPI has no full-text search API - only exact package name lookups
- Homebrew formula download can be slow on first run (~5MB JSON)
- CLI-based searches depend on local package indexes being up-to-date
- No caching for CLI results or API responses (except Homebrew)
- Network timeouts may cause incomplete results

---

## 9. Command Execution Framework

### RunResult Data Model
```python
@dataclass
class RunResult:
    ok: bool          # True if exit code == 0
    code: int         # Process exit code (or -1 for exceptions)
    out: str          # stdout content
    err: str          # stderr content
```

### Command Execution Function
```python
def run_command(cmd: Union[List[str], str], timeout=300, retries=1, show_progress=False, shell=False, cwd=None) -> RunResult:
    """Enhanced subprocess execution with progress tracking and retries"""
    
    cmd_str = ' '.join(cmd) if isinstance(cmd, list) else cmd
    if LOG.verbose:
        cprint(f"Running: {cmd_str}", "INFO")
    
    for attempt in range(retries + 1):
        if attempt > 0:
            cprint(f"Retry attempt {attempt}/{retries}", "WARNING")
            time.sleep(2)
        
        try:
            # Process creation with proper configuration
            if shell:
                process = subprocess.Popen(
                    cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
                    text=True, cwd=cwd, bufsize=1, universal_newlines=True
                )
            else:
                process = subprocess.Popen(
                    cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
                    text=True, cwd=cwd, bufsize=1, universal_newlines=True
                )
            
            # Optional progress indication
            if show_progress and not LOG.json_mode:
                progress_thread = threading.Thread(target=_show_progress_dots, args=(process,))
                progress_thread.daemon = True
                progress_thread.start()
            
            # Wait for completion with timeout
            try:
                stdout, stderr = process.communicate(timeout=timeout)
            except subprocess.TimeoutExpired:
                process.kill()
                stdout, stderr = process.communicate()
                return RunResult(False, -1, stdout, f"Command timed out after {timeout} seconds")
            
            result = RunResult(
                ok=(process.returncode == 0),
                code=process.returncode,
                out=stdout,
                err=stderr
            )
            
            if LOG.verbose and not result.ok:
                cprint(f"Command failed with exit code {result.code}", "ERROR")
                if result.err:
                    cprint(f"Error output: {result.err[:500]}", "ERROR")
            
            return result
            
        except Exception as e:
            if attempt == retries:
                return RunResult(False, -1, "", f"Exception: {str(e)}")
            time.sleep(1)
    
    return RunResult(False, -1, "", "All retry attempts failed")
```

### Progress Indicator Helper
```python
def _show_progress_dots(process):
    """Shows animated spinner while process is running"""
    spinner_chars = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"]
    i = 0
    while process.poll() is None:
        if process.poll() is not None:
            return
        sys.stdout.write(f"\r{spinner_chars[i % len(spinner_chars)]} Working...")
        sys.stdout.flush()
        time.sleep(0.1)
        i += 1
    sys.stdout.write("\r" + " " * 20 + "\r")  # Clear the line
```

**Usage Patterns:**
- Package installation/removal commands (long-running, with progress)
- System cleanup operations
- Manager update commands  
- Network testing utilities

**Error Handling:**
- Timeout handling with process termination
- Retry logic with exponential backoff
- Exception capture for system-level errors
- Detailed logging in verbose mode

---

## 10. Package Manager Detection & Commands

### Python Command Detection
```python
def _get_python_commands() -> List[List[str]]:
    """Discovers available Python interpreters in order of preference"""
    commands = []
    
    # Current interpreter first (most reliable)
    if sys.executable:
        commands.append([sys.executable])
    
    # Common Python commands
    for cmd in ["python3", "python", "py"]:
        if shutil.which(cmd):
            commands.append([cmd])
    
    return commands
```

### Install Command Builders

Each manager has a dedicated function that returns the command list for installing a package:

```python
def _pip_install(pkg: str) -> List[str]:
    """Builds pip install command using best available Python"""
    python_cmds = _get_python_commands()
    for cmd in python_cmds:
        if shutil.which(cmd[0]):
            return cmd + ["-m", "pip", "install", "--user", pkg]
    return [sys.executable, "-m", "pip", "install", "--user", pkg]

def _npm_install(pkg: str) -> List[str]:
    return ["npm", "install", "-g", pkg]

def _apt_install(pkg: str) -> List[str]:
    return ["sudo", "apt", "install", "-y", pkg]

def _dnf_install(pkg: str) -> List[str]:
    return ["sudo", "dnf", "install", "-y", pkg]

def _yum_install(pkg: str) -> List[str]:
    return ["sudo", "yum", "install", "-y", pkg]

def _pacman_install(pkg: str) -> List[str]:
    return ["sudo", "pacman", "-S", "--noconfirm", pkg]

def _zypper_install(pkg: str) -> List[str]:
    return ["sudo", "zypper", "--non-interactive", "install", pkg]

def _apk_install(pkg: str) -> List[str]:
    return ["sudo", "apk", "add", pkg]

def _brew_install(pkg: str) -> List[str]:
    return ["brew", "install", pkg]

def _choco_install(pkg: str) -> List[str]:
    return ["choco", "install", "-y", pkg]

def _winget_install(pkg: str) -> List[str]:
    return ["winget", "install", "--silent", "--accept-package-agreements", "--accept-source-agreements", pkg]

def _snap_install(pkg: str) -> List[str]:
    return ["sudo", "snap", "install", pkg]

def _flatpak_install(pkg: str) -> List[str]:
    return ["flatpak", "install", "-y", pkg]
```

### Remove Command Builders

```python
def _pip_remove(pkg: str) -> List[str]:
    python_cmds = _get_python_commands()
    for cmd in python_cmds:
        if shutil.which(cmd[0]):
            return cmd + ["-m", "pip", "uninstall", "-y", pkg]
    return [sys.executable, "-m", "pip", "uninstall", "-y", pkg]

def _npm_remove(pkg: str) -> List[str]:
    return ["npm", "uninstall", "-g", pkg]

def _brew_remove(pkg: str) -> List[str]:
    return ["brew", "uninstall", pkg]

def _apt_remove(pkg: str) -> List[str]:
    return ["sudo", "apt", "remove", "-y", pkg]

def _dnf_remove(pkg: str) -> List[str]:
    return ["sudo", "dnf", "remove", "-y", pkg]

def _yum_remove(pkg: str) -> List[str]:
    return ["sudo", "yum", "remove", "-y", pkg]

def _pacman_remove(pkg: str) -> List[str]:
    return ["sudo", "pacman", "-R", "--noconfirm", pkg]

def _snap_remove(pkg: str) -> List[str]:
    return ["sudo", "snap", "remove", pkg]

def _flatpak_remove(pkg: str) -> List[str]:
    return ["flatpak", "uninstall", "-y", pkg]
```

### Manager Command Mappings

```python
MANAGER_INSTALL_HANDLERS: Dict[str, callable] = {
    "pip": _pip_install, "npm": _npm_install, "apt": _apt_install, 
    "dnf": _dnf_install, "yum": _yum_install, "pacman": _pacman_install, 
    "zypper": _zypper_install, "apk": _apk_install, "brew": _brew_install,
    "choco": _choco_install, "winget": _winget_install, "snap": _snap_install, 
    "flatpak": _flatpak_install,
}

MANAGER_REMOVE_HANDLERS = {
    "pip": _pip_remove, "npm": _npm_remove, "brew": _brew_remove,
    "apt": _apt_remove, "dnf": _dnf_remove, "yum": _yum_remove,
    "pacman": _pacman_remove, "snap": _snap_remove, "flatpak": _flatpak_remove,
}
```

### Manager Detection Logic

```python
def _detect_installed_managers() -> Dict[str, bool]:
    """Detects which package managers are available on the system"""
    available = {}
    
    for name, fn in MANAGER_INSTALL_HANDLERS.items():
        if name == "pip":
            # Special handling for pip - check if any Python/pip combination works
            python_cmds = _get_python_commands()
            available[name] = any(shutil.which(cmd[0]) for cmd in python_cmds if cmd)
        else:
            # Standard binary availability check
            available[name] = shutil.which(name) is not None
    
    return available
```

### Package Type Heuristics

```python
def _looks_like_python_pkg(pkg: str) -> bool:
    """Heuristics for identifying Python packages"""
    python_indicators = ["==", ">=", "<=", "~=", "!=", "[", "]"]
    python_common = ["py", "django", "flask", "numpy", "pandas", "requests", "boto3", "tensorflow", "torch"]
    
    # Version specifiers indicate Python package
    if any(indicator in pkg for indicator in python_indicators):
        return True
    
    # Common Python package prefixes/names
    pkg_lower = pkg.lower()
    if any(pkg_lower.startswith(prefix) for prefix in python_common):
        return True
    
    return False

def _looks_like_npm_pkg(pkg: str) -> bool:
    """Heuristics for identifying NPM packages"""
    npm_indicators = pkg.startswith("@")  # Scoped packages
    npm_common = ["express", "react", "vue", "angular", "typescript", "eslint", "webpack", "lodash", "axios"]
    
    if npm_indicators:
        return True
    
    pkg_lower = pkg.lower()
    if pkg_lower in npm_common:
        return True
    
    return False
```

### OS-Specific Manager Prioritization

```python
def _os_type() -> str:
    """Simplified OS classification for manager selection"""
    s = platform.system().lower()
    if s.startswith("win"): return "windows"
    if s == "darwin": return "macos"
    if s == "linux": return "linux"
    return "unknown"

def _system_manager_priority() -> List[str]:
    """Returns prioritized list of system package managers by OS"""
    ot = _os_type()
    
    if ot == "macos": 
        return ["brew", "snap", "flatpak"]
    elif ot == "windows": 
        return ["winget", "choco"]
    elif ot == "linux":
        # Detect specific Linux distribution managers
        linux_managers = [
            ("apt", ["apt", "apt-get"]),      # Debian/Ubuntu
            ("dnf", ["dnf"]),                 # Fedora
            ("yum", ["yum"]),                 # RHEL/CentOS
            ("pacman", ["pacman"]),           # Arch Linux
            ("zypper", ["zypper"]),           # openSUSE
            ("apk", ["apk"])                  # Alpine Linux
        ]
        
        for manager, commands in linux_managers:
            if any(shutil.which(cmd) for cmd in commands):
                return [manager, "snap", "flatpak"]
        
        # Fallback to universal package managers
        return ["snap", "flatpak"]
    
    return ["snap", "flatpak"]

def _ordered_install_manager_candidates(pkg: str, installed: Dict[str, bool]) -> List[str]:
    """Generates prioritized manager list for package installation"""
    prefs: List[str] = []
    
    # Heuristic-based prioritization
    if _looks_like_python_pkg(pkg) and installed.get("pip"):
        prefs.append("pip")
    if _looks_like_npm_pkg(pkg) and installed.get("npm"):
        prefs.append("npm")
    
    # Add system-specific managers
    for manager in _system_manager_priority():
        if installed.get(manager) and manager not in prefs:
            prefs.append(manager)
    
    # Add remaining available managers
    for manager, is_installed in installed.items():
        if is_installed and manager not in prefs:
            prefs.append(manager)
    
    return prefs

def _manager_human(name: str) -> str:
    """Human-readable manager names for display"""
    names = {
        "pip": "Python (pip)", "npm": "Node.js (npm)", "apt": "APT", "dnf": "DNF", 
        "yum": "YUM", "pacman": "Pacman", "zypper": "Zypper", "apk": "APK", 
        "brew": "Homebrew", "choco": "Chocolatey", "winget": "Winget", 
        "snap": "Snap", "flatpak": "Flatpak",
    }
    return names.get(name, name.title())

def _extract_package_version(output: str, manager: str) -> str:
    """Extracts version information from installation output"""
    try:
        if manager == "pip":
            # "Successfully installed package-version"
            match = re.search(r"Successfully installed .* (\S+)-(\d+\.\d+\.\d+)", output)
            if match:
                return match.group(2)
        elif manager == "npm":
            # "@version" in npm output
            match = re.search(r"@(\d+\.\d+\.\d+)", output)
            if match:
                return match.group(1)
        elif manager in ["apt", "dnf", "yum"]:
            # Generic version pattern
            match = re.search(r"(\d+\.\d+\.\d+)", output)
            if match:
                return match.group(1)
    except:
        pass
    return "installed"  # Fallback for failed parsing
```

**Key Design Notes:**
- Pip uses `--user` flag for user-local installations (avoids sudo)
- System managers (apt, dnf, etc.) require sudo for installation
- Windows managers (winget, choco) use different flag patterns
- Heuristics help but aren't foolproof - ambiguous package names may try wrong managers first
- Manager detection is binary (available/unavailable) with no version or capability checking

---

## 11. Installation & Removal Orchestration

### Installation Flow

```python
def install_package(pkg: str, preferred_manager: Optional[str] = None) -> Tuple[bool, List[Tuple[str, RunResult]]]:
    """Orchestrates package installation across multiple managers with fallback"""
    
    cprint(f"Preparing to install: {pkg}", "INFO")
    installed = _detect_installed_managers()
    
    if not any(installed.values()):
        cprint("No supported package managers are available on this system.", "ERROR")
        return (False, [])
    
    attempts: List[Tuple[str, RunResult]] = []
    candidates = _ordered_install_manager_candidates(pkg, installed)

    # Handle preferred manager
    if preferred_manager:
        pm = preferred_manager.lower()
        if pm in MANAGER_INSTALL_HANDLERS and installed.get(pm):
            # Move preferred manager to front of candidates
            candidates = [pm] + [m for m in candidates if m != pm]
        else:
            available_managers = [m for m, avail in installed.items() if avail]
            cprint(f"Warning: --manager '{preferred_manager}' not available. Available: {', '.join(available_managers)}", "WARNING")

    if not candidates:
        cprint("No package managers available for installation.", "ERROR")
        return (False, [])

    # Show installation plan
    cprint("Installation plan:", "CYAN")
    for i, m in enumerate(candidates, 1):
        cprint(f"  {i}. {_manager_human(m)}", "MUTED")

    # Try each manager in sequence
    for i, manager in enumerate(candidates, 1):
        cmd_builder = MANAGER_INSTALL_HANDLERS.get(manager)
        if not cmd_builder:
            continue
            
        try:
            cmd = cmd_builder(pkg)
            cprint(f"Attempt {i}/{len(candidates)}: Installing via {_manager_human(manager)}...", "INFO")
            
            # Execute with extended timeout for compilations
            res = run_command(cmd, timeout=1800, retries=0, show_progress=True)
            attempts.append((manager, res))
            
            if res.ok:
                # Extract and record successful installation
                version = _extract_package_version(res.out, manager)
                package_db.add_package(pkg, version, manager, ' '.join(cmd))
                
                cprint(f"Successfully installed '{pkg}' via {_manager_human(manager)}", "SUCCESS")
                return (True, attempts)
            else:
                # Show helpful error summary
                err_msg = (res.err or res.out).strip()
                if err_msg:
                    error_lines = err_msg.splitlines()
                    relevant_error = error_lines[-1] if error_lines else "Unknown error"
                    if len(relevant_error) > 180:
                        relevant_error = relevant_error[:177] + "..."
                    cprint(f"{_manager_human(manager)} failed: {relevant_error}", "WARNING")
                else:
                    cprint(f"{_manager_human(manager)} failed with no error message", "WARNING")
                    
        except Exception as e:
            err_result = RunResult(False, -1, "", str(e))
            attempts.append((manager, err_result))
            cprint(f"{_manager_human(manager)} failed with exception: {str(e)}", "WARNING")

    cprint(f"Failed to install '{pkg}' with all available managers.", "ERROR")
    return (False, attempts)
```

### Removal Flow

```python
def remove_package(pkg: str, manager: Optional[str] = None) -> Tuple[bool, List[Tuple[str, RunResult]]]:
    """Orchestrates package removal with optional manager specification"""
    
    cprint(f"Preparing to remove: {pkg}", "INFO")
    installed = _detect_installed_managers()
    
    if not any(installed.values()):
        cprint("No supported package managers are available on this system.", "ERROR")
        return (False, [])
    
    attempts: List[Tuple[str, RunResult]] = []
    
    if manager:
        # Specific manager requested
        if manager.lower() in MANAGER_REMOVE_HANDLERS and installed.get(manager.lower()):
            candidates = [manager.lower()]
        else:
            cprint(f"Manager '{manager}' not available for package removal", "ERROR")
            return (False, [])
    else:
        # Try managers in priority order
        candidates = _ordered_install_manager_candidates(pkg, installed)
        # Filter to only those that support removal
        candidates = [m for m in candidates if m in MANAGER_REMOVE_HANDLERS]

    if not candidates:
        cprint("No package managers available for package removal.", "ERROR")
        return (False, [])

    # Attempt removal with each manager
    for mgr in candidates:
        cmd_builder = MANAGER_REMOVE_HANDLERS.get(mgr)
        if not cmd_builder:
            continue
            
        try:
            cmd = cmd_builder(pkg)
            cprint(f"Attempting removal via {_manager_human(mgr)}...", "INFO")
            
            res = run_command(cmd, timeout=600, retries=0, show_progress=True)
            attempts.append((mgr, res))
            
            if res.ok:
                # Remove from tracking database
                package_db.remove_package(pkg, mgr)
                
                cprint(f"Removed '{pkg}' via {_manager_human(mgr)}", "SUCCESS")
                return (True, attempts)
            else:
                err_msg = (res.err or res.out).strip()
                if err_msg:
                    error_lines = err_msg.splitlines()
                    relevant_error = error_lines[-1] if error_lines else "Unknown error"
                    if len(relevant_error) > 180:
                        relevant_error = relevant_error[:177] + "..."
                    cprint(f"{_manager_human(mgr)} failed: {relevant_error}", "WARNING")
                else:
                    cprint(f"{_manager_human(mgr)} failed with no error message", "WARNING")
                    
        except Exception as e:
            err_result = RunResult(False, -1, "", str(e))
            attempts.append((mgr, err_result))
            cprint(f"{_manager_human(mgr)} failed with exception: {str(e)}", "WARNING")

    cprint(f"Failed to remove '{pkg}' with all available managers.", "ERROR")
    return (False, attempts)
```

**Key Behaviors:**
- Installation tries managers in heuristic-based priority order
- Preferred manager (via `--manager` flag) is moved to front of attempt list
- Each attempt is recorded with full command output for debugging
- Database is only updated on successful operations
- Error messages are truncated for readability but preserved in attempts list
- Removal without manager specification tries same priority order as installation
- Long timeout (1800s) for installations to handle large compilations
- Progress indicators shown for long-running operations

**Edge Cases:**
- Interactive installers requiring user input will hang (no TTY handling)
- Some managers may install different versions than requested (latest vs specific)
- Database tracking may become inconsistent if packages are installed/removed outside CrossFire
- Network failures during download can leave partial installations

---

## 12. System Maintenance & Cleanup

### Cleanup System

```python
def cleanup_system() -> Dict[str, Dict[str, str]]:
    """Cleans package manager caches and temporary files across all available managers"""
    
    cprint("Starting comprehensive system cleanup...", "INFO")
    results = {}
    installed = _detect_installed_managers()
    
    # Cleanup commands for each manager
    cleanup_commands = {
        "pip": [sys.executable, "-m", "pip", "cache", "purge"],
        "npm": ["npm", "cache", "clean", "--force"],
        "brew": ["brew", "cleanup", "--prune=all"],
        "apt": "sudo apt autoremove -y && sudo apt autoclean",  # Shell command
        "dnf": ["sudo", "dnf", "clean", "all"],
        "yum": ["sudo", "yum", "clean", "all"],
        "pacman": ["sudo", "pacman", "-Sc", "--noconfirm"],
    }
    
    available_cleanups = [(mgr, cmd) for mgr, cmd in cleanup_commands.items() if installed.get(mgr)]
    
    if not available_cleanups:
        cprint("No package managers found to clean up.", "WARNING")
        return results
    
    progress = ProgressBar(len(available_cleanups), "Cleanup progress", "managers")
    
    for manager, cmd in available_cleanups:
        try:
            cprint(f"Cleaning {_manager_human(manager)}...", "INFO")
            use_shell = isinstance(cmd, str)  # Shell commands vs. list commands
            result = run_command(cmd, timeout=300, shell=use_shell)
            
            if result.ok:
                results[manager] = {"ok": "true", "msg": "Cleanup successful"}
                cprint(f"{_manager_human(manager)}: Cleanup successful", "SUCCESS")
            else:
                results[manager] = {"ok": "false", "msg": result.err or "Cleanup failed"}
                cprint(f"{_manager_human(manager)}: Cleanup failed", "WARNING")
                
        except Exception as e:
            results[manager] = {"ok": "false", "msg": f"Exception: {e}"}
            cprint(f"{_manager_human(manager)}: Exception during cleanup: {e}", "WARNING")
        finally:
            progress.update(1)
    
    progress.finish()
    
    # Summary statistics
    successful = sum(1 for r in results.values() if r.get("ok") == "true")
    total = len(results)
    cprint(f"Cleanup complete: {successful}/{total} managers cleaned successfully", 
           "SUCCESS" if successful > 0 else "WARNING")
    
    return results
```

### Manager Update System

```python
def _update_manager(manager: str) -> Tuple[str, bool, str]:
    """Updates a specific package manager to latest version"""
    manager = manager.lower()
    
    # Update commands for each manager
    update_commands = {
        "pip": [sys.executable, "-m", "pip", "install", "--upgrade", "pip"],
        "npm": ["npm", "update", "-g", "npm"],
        "brew": ["brew", "update", "&&", "brew", "upgrade"],  # BUG: This will fail!
        "apt": "sudo apt update && sudo apt upgrade -y",      # Shell command
        "dnf": ["sudo", "dnf", "update", "-y"],
        "yum": ["sudo", "yum", "update", "-y"],
        "pacman": ["sudo", "pacman", "-Syu", "--noconfirm"],
        "snap": ["sudo", "snap", "refresh"],
        "flatpak": ["flatpak", "update", "-y"],
    }
    
    cmd = update_commands.get(manager)
    if not cmd:
        return (manager, False, f"Update not supported for {manager}")
    
    try:
        use_shell = isinstance(cmd, str)
        result = run_command(cmd, timeout=600, show_progress=True, shell=use_shell)
        
        if result.ok:
            return (manager, True, "Update successful")
        else:
            return (manager, False, result.err or "Update failed")
    except Exception as e:
        return (manager, False, f"Exception: {e}")

def _update_all_managers() -> Dict[str, Dict[str, str]]:
    """Updates all available package managers"""
    installed = _detect_installed_managers()
    available_managers = [mgr for mgr, avail in installed.items() if avail]
    
    if not available_managers:
        return {}
    
    results = {}
    progress = ProgressBar(len(available_managers), "Updating managers", "managers")
    
    for manager in available_managers:
        name, ok, msg = _update_manager(manager)
        results[name] = {"ok": str(ok).lower(), "msg": msg}
        
        cprint(f"{_manager_human(name)}: {msg}", "SUCCESS" if ok else "WARNING")
        progress.update()
    
    progress.finish()
    return results
```

**Critical Bug Identified:**
The brew update command is defined as:
```python
"brew": ["brew", "update", "&&", "brew", "upgrade"]
```
This passes `"&&"` as a literal argument to the `brew` command, which will fail. It should either be:
```python
"brew": "brew update && brew upgrade"  # Shell command (shell=True)
```
or split into two separate commands.

---

## 13. Self-Update System

### File Download with Progress

```python
def download_file_with_progress(url: str, dest_path: Path, expected_hash: Optional[str] = None) -> bool:
    """Downloads file with progress tracking and optional SHA256 verification"""
    try:
        cprint(f"Downloading from: {url}", "INFO")
        
        # Prepare HTTP request with user agent
        request = urllib.request.Request(url)
        request.add_header('User-Agent', f'CrossFire/{__version__}')
        
        with urllib.request.urlopen(request, timeout=30) as response:
            total_size = int(response.info().get('Content-Length', 0))
            
            if total_size == 0:
                cprint("Warning: Cannot determine file size", "WARNING")
                total_size = 1024 * 1024  # Assume 1MB for progress bar
            
            # Setup progress tracking and hashing
            progress = ProgressBar(total_size, "Download", "B")
            downloaded = 0
            hash_sha256 = hashlib.sha256()
            
            # Create destination directory
            dest_path.parent.mkdir(parents=True, exist_ok=True)
            
            with open(dest_path, 'wb') as f:
                while True:
                    chunk = response.read(32768)  # 32KB chunks
                    if not chunk:
                        break
                    
                    f.write(chunk)
                    downloaded += len(chunk)
                    hash_sha256.update(chunk)
                    progress.update(len(chunk))
            
            progress.finish()
            
            # Verify hash if provided
            if expected_hash:
                actual_hash = hash_sha256.hexdigest()
                if actual_hash.lower() != expected_hash.lower():
                    cprint(f"Hash verification failed!", "ERROR")
                    cprint(f"Expected: {expected_hash}", "ERROR")
                    cprint(f"Actual:   {actual_hash}", "ERROR")
                    dest_path.unlink()  # Remove corrupted file
                    return False
                else:
                    cprint("Hash verification successful", "SUCCESS")
            
            cprint(f"Downloaded {downloaded / 1024 / 1024:.1f} MB successfully", "SUCCESS")
            return True
            
    except Exception as e:
        cprint(f"Download failed: {e}", "ERROR")
        if dest_path.exists():
            dest_path.unlink()  # Clean up partial download
        return False
```

### Self-Update Logic

```python
def cross_update(url: str, verify_sha256: Optional[str] = None) -> bool:
    """Self-updates CrossFire from remote URL with verification"""
    cprint(f"Starting CrossFire self-update...", "INFO")
    
    try:
        # Download to temporary file in cache
        temp_file = CROSSFIRE_CACHE / f"crossfire_update_{int(time.time())}.py"
        
        if not download_file_with_progress(url, temp_file, verify_sha256):
            return False
        
        # Determine current script path
        current_script = Path(sys.argv[0]).resolve()
        
        # Create backup of current version
        backup_path = current_script.with_suffix('.py.backup')
        if current_script.exists():
            shutil.copy2(current_script, backup_path)
            cprint(f"Backup created: {backup_path}", "INFO")
        
        # Replace current script atomically
        shutil.copy2(temp_file, current_script)
        
        # Set executable permissions on Unix systems
        if OS_NAME != "Windows":
            current_script.chmod(current_script.stat().st_mode | stat.S_IEXEC)
        
        # Clean up temporary file
        temp_file.unlink()
        
        cprint(f"CrossFire updated successfully!", "SUCCESS")
        cprint(f"Please restart CrossFire to use the new version", "INFO")
        return True
        
    except Exception as e:
        cprint(f"Update failed: {e}", "ERROR")
        return False
```

**Security & Risk Analysis:**
- Downloads over HTTPS but no certificate pinning
- Optional SHA256 verification (not enforced by default)
- No GPG signature verification
- Replaces running script file (may fail if no write permissions)
- Creates backup but no automatic rollback on failure
- User-Agent header identifies requests as coming from CrossFire

**Potential Issues:**
- Update may fail if script is running from system location without write access
- No verification that downloaded file is valid Python code
- Backup strategy doesn't handle multiple update attempts
- Network interruptions can corrupt downloads (mitigated by hash verification)

---

## 14. System Information & Health Checks

### System Information Collection

```python
def get_system_info() -> Dict[str, Any]:
    """Collects comprehensive system information for diagnostics"""
    info = {
        "crossfire_version": __version__,
        "system": {
            "os": OS_NAME,
            "distribution": DISTRO_NAME,
            "version": DISTRO_VERSION,
            "architecture": ARCH,
            "python_version": platform.python_version(),
        },
        "package_managers": _detect_installed_managers(),
        "crossfire_data": {
            "directory": str(CROSSFIRE_DIR),
            "database": str(CROSSFIRE_DB),
            "cache": str(CROSSFIRE_CACHE),
        }
    }
    
    # Calculate cache directory size
    try:
        cache_size = sum(f.stat().st_size for f in CROSSFIRE_CACHE.rglob('*') if f.is_file())
        info["crossfire_data"]["cache_size_mb"] = round(cache_size / 1024 / 1024, 2)
    except:
        info["crossfire_data"]["cache_size_mb"] = 0
    
    return info
```

### Comprehensive Health Check

```python
def health_check() -> Dict[str, Any]:
    """Runs comprehensive system health diagnostics"""
    cprint("Running system health check...", "INFO")
    
    results = {
        "overall_status": "healthy",
        "checks": {},
        "recommendations": []
    }
    
    # Package Manager Availability Check
    managers = _detect_installed_managers()
    manager_count = sum(1 for available in managers.values() if available)
    
    results["checks"]["package_managers"] = {
        "status": "good" if manager_count >= 2 else "warning" if manager_count >= 1 else "error",
        "available_count": manager_count,
        "total_supported": len(managers),
        "available": [m for m, avail in managers.items() if avail]
    }
    
    if manager_count == 0:
        results["recommendations"].append("Install at least one package manager (pip, npm, brew, apt, etc.)")
        results["overall_status"] = "unhealthy"
    elif manager_count == 1:
        results["recommendations"].append("Consider installing additional package managers for better coverage")
    
    # Internet Connectivity Check
    try:
        urllib.request.urlopen("https://google.com", timeout=10)
        results["checks"]["internet"] = {"status": "good", "message": "Internet connection available"}
    except:
        results["checks"]["internet"] = {"status": "error", "message": "No internet connection"}
        results["recommendations"].append("Check your internet connection")
        results["overall_status"] = "unhealthy"
    
    # Database Integrity Check
    try:
        installed_packages = package_db.get_installed_packages()
        results["checks"]["database"] = {
            "status": "good",
            "installed_packages": len(installed_packages),
            "database_path": str(CROSSFIRE_DB)
        }
    except Exception as e:
        results["checks"]["database"] = {
            "status": "error", 
            "message": f"Database error: {e}"
        }
        results["recommendations"].append("Database may be corrupted - consider clearing CrossFire data")
    
    # Disk Space Check
    try:
        free_space = shutil.disk_usage(CROSSFIRE_DIR).free
        free_space_gb = free_space / (1024**3)
        
        if free_space_gb < 0.1:  # Less than 100MB
            results["checks"]["disk_space"] = {
                "status": "error",
                "free_space_gb": round(free_space_gb, 2)
            }
            results["recommendations"].append("Very low disk space - clean up files")
            results["overall_status"] = "unhealthy"
        elif free_space_gb < 1:  # Less than 1GB
            results["checks"]["disk_space"] = {
                "status": "warning",
                "free_space_gb": round(free_space_gb, 2)
            }
            results["recommendations"].append("Low disk space - consider cleanup")
        else:
            results["checks"]["disk_space"] = {
                "status": "good",
                "free_space_gb": round(free_space_gb, 2)
            }
    except:
        results["checks"]["disk_space"] = {"status": "unknown", "message": "Could not check disk space"}
    
    # Determine final overall status
    if any(check.get("status") == "error" for check in results["checks"].values()):
        results["overall_status"] = "unhealthy"
    elif any(check.get("status") == "warning" for check in results["checks"].values()):
        results["overall_status"] = "needs_attention"
    
    # Display results in console mode
    if not LOG.json_mode:
        status_colors = {"healthy": "SUCCESS", "needs_attention": "WARNING", "unhealthy": "ERROR"}
        status_icons = {"healthy": "Healthy", "needs_attention": "Needs Attention", "unhealthy": "Unhealthy"}
        
        overall_status = results["overall_status"]
        cprint(f"Overall Status: {status_icons[overall_status].upper()}", status_colors[overall_status])
        
        for check_name, check_result in results["checks"].items():
            status = check_result.get("status", "unknown")
            icon = {"good": "Good", "warning": "Warning", "error": "Error", "unknown": "Unknown"}[status]
            cprint(f"  {check_name.replace('_', ' ').title()}: {icon}", 
                   {"good": "SUCCESS", "warning": "WARNING", "error": "ERROR", "unknown": "INFO"}[status])
        
        if results["recommendations"]:
            cprint("\nRecommendations:", "CYAN")
            for i, rec in enumerate(results["recommendations"], 1):
                cprint(f"  {i}. {rec}", "INFO")
    
    return results
```

---

## 15. Setup & Launcher Installation

### PATH Management

```python
def add_to_path_safely() -> bool:
    """Adds CrossFire to system PATH across different shells"""
    cprint("Adding CrossFire to PATH...", "INFO")
    
    try:
        # Determine installation directory by platform
        if OS_NAME == "Windows":
            install_dir = Path.home() / "AppData" / "Local" / "CrossFire"
        else:
            install_dir = Path.home() / ".local" / "bin"
        
        install_dir.mkdir(parents=True, exist_ok=True)
        
        # Shell configuration files to update (Unix/Linux/macOS only)
        shell_configs = []
        if OS_NAME != "Windows":
            home = Path.home()
            shell_configs = [
                home / ".bashrc",
                home / ".zshrc", 
                home / ".profile",
                home / ".bash_profile"
            ]
        
        path_entry = str(install_dir)
        
        for config_file in shell_configs:
            if config_file.exists():
                try:
                    # Read current content
                    content = config_file.read_text()
                    
                    # Check if already added (avoid duplicates)
                    if f'export PATH="{path_entry}:$PATH"' in content:
                        continue
                    
                    # Append PATH export
                    with config_file.open('a') as f:
                        f.write(f'\n# Added by CrossFire\nexport PATH="{path_entry}:$PATH"\n')
                    
                    cprint(f"Updated {config_file.name}", "SUCCESS")
                except Exception as e:
                    cprint(f"Could not update {config_file.name}: {e}", "WARNING")
        
        return True
        
    except Exception as e:
        cprint(f"Failed to add to PATH: {e}", "ERROR")
        return False

def install_launcher(target_dir: Optional[str] = None) -> Optional[str]:
    """Installs CrossFire launcher globally with optional custom directory"""
    cprint("Installing CrossFire launcher...", "INFO")

    try:
        if target_dir:
            install_dir = Path(target_dir).expanduser().resolve()
        elif OS_NAME == "Windows":
            install_dir = Path.home() / "AppData" / "Local" / "CrossFire"
        else:
            install_dir = Path.home() / ".local" / "bin"

        install_dir.mkdir(parents=True, exist_ok=True)

        launcher_name = "crossfire.exe" if OS_NAME == "Windows" else "crossfire"
        launcher_path = install_dir / launcher_name

        # Copy current script to launcher location
        current_script = Path(__file__).resolve()
        shutil.copy2(current_script, launcher_path)

        # Set executable permissions on Unix systems
        if OS_NAME != "Windows":
            launcher_path.chmod(launcher_path.stat().st_mode | stat.S_IEXEC)

        cprint(f"Launcher installed to: {launcher_path}", "SUCCESS")
        return str(launcher_path)
    except Exception as e:
        cprint(f"Failed to install launcher: {e}", "ERROR")
    return None
```

**Platform-Specific Behavior:**
- **Unix/Linux/macOS**: Installs to `~/.local/bin/crossfire`, updates shell rc files
- **Windows**: Installs to `%LOCALAPPDATA%\CrossFire\crossfire.exe`, no PATH modification
- Shell rc file updates are append-only and check for duplicates
- No automatic shell reload or sourcing

---

## 16. Package Management Features

### Package Listing & Statistics

```python
def show_installed_packages():
    """Displays packages installed via CrossFire with detailed formatting"""
    packages = package_db.get_installed_packages()
    
    if LOG.json_mode:
        print(json.dumps(packages, indent=2, default=str))
        return
    
    if not packages:
        cprint("No packages have been installed via CrossFire yet.", "INFO")
        cprint("Packages installed directly via other managers won't appear here.", "MUTED")
        return
    
    cprint(f"Packages Installed via CrossFire ({len(packages)})", "SUCCESS")
    cprint("=" * 70, "CYAN")
    
    # Group packages by manager
    by_manager = {}
    for pkg in packages:
        manager = pkg['manager']
        if manager not in by_manager:
            by_manager[manager] = []
        by_manager[manager].append(pkg)
    
    for manager in sorted(by_manager.keys()):
        pkgs = by_manager[manager]
        cprint(f"\n{_manager_human(manager)} ({len(pkgs)} packages)", "INFO")
        
        for i, pkg in enumerate(sorted(pkgs, key=lambda x: x['install_date'], reverse=True), 1):
            install_date = pkg['install_date'][:10] if pkg['install_date'] else 'unknown'
            version = pkg.get('version', 'unknown')
            
            cprint(f"  {i:2d}. {pkg['name']} "
                   f"v{version} "
                   f"(installed {install_date})", "SUCCESS")
    
    cprint(f"\nRemove with: crossfire -r <package_name>", "INFO")

def get_package_statistics() -> Dict[str, Any]:
    """Generates detailed package installation statistics"""
    packages = package_db.get_installed_packages()
    managers = _detect_installed_managers()
    
    stats = {
        "total_packages": len(packages),
        "packages_by_manager": {},
        "recent_installations": [],
        "available_managers": sum(1 for avail in managers.values() if avail),
        "total_supported_managers": len(managers)
    }
    
    # Count packages by manager
    for pkg in packages:
        manager = pkg['manager']
        stats["packages_by_manager"][manager] = stats["packages_by_manager"].get(manager, 0) + 1
    
    # Find recent installations (last 7 days)
    try:
        week_ago = datetime.now() - timedelta(days=7)
        for pkg in packages:
            if pkg['install_date']:
                install_date = datetime.fromisoformat(pkg['install_date'].replace('Z', '+00:00'))
                if install_date > week_ago:
                    stats["recent_installations"].append({
                        "name": pkg['name'],
                        "manager": pkg['manager'],
                        "date": pkg['install_date']
                    })
    except:
        pass  # Handle date parsing errors gracefully
    
    return stats

def show_statistics():
    """Displays detailed statistics with ASCII bar charts"""
    stats = get_package_statistics()
    
    if LOG.json_mode:
        print(json.dumps(stats, indent=2, default=str))
        return
    
    cprint(f"CrossFire Statistics", "SUCCESS")
    cprint("=" * 50, "CYAN")
    
    # Overview section
    cprint(f"\nOverview", "INFO")
    cprint(f"  Total packages installed via CrossFire: {stats['total_packages']}", "SUCCESS")
    cprint(f"  Available package managers: {stats['available_managers']}/{stats['total_supported_managers']}", "SUCCESS")
    
    # Packages by manager with ASCII bar chart
    if stats["packages_by_manager"]:
        cprint(f"\nPackages by Manager", "INFO")
        for manager, count in sorted(stats["packages_by_manager"].items(), key=lambda x: x[1], reverse=True):
            percentage = (count / stats['total_packages']) * 100
            bar_length = int(percentage / 5)  # Scale to 20 characters max
            bar = "█" * bar_length + "░" * (20 - bar_length)
            cprint(f"  {_manager_human(manager):15} │{bar}│ {count:3d} ({percentage:4.1f}%)", "SUCCESS")
    
    # Recent activity
    if stats["recent_installations"]:
        cprint(f"\nRecent Installations (Last 7 Days)", "INFO")
        for pkg in stats["recent_installations"][:5]:  # Show last 5
            date = pkg['date'][:10] if pkg['date'] else 'unknown'
            cprint(f"  • {pkg['name']} via {_manager_human(pkg['manager'])} on {date}", "SUCCESS")
        
        if len(stats["recent_installations"]) > 5:
            cprint(f"  ... and {len(stats['recent_installations']) - 5} more", "MUTED")
```

### Import/Export Functionality

```python
def bulk_install_from_file(file_path: str) -> Dict[str, Any]:
    """Installs packages from requirements file with detailed progress tracking"""
    cprint(f"Installing packages from: {file_path}", "INFO")
    
    try:
        path = Path(file_path)
        if not path.exists():
            cprint(f"File not found: {file_path}", "ERROR")
            return {"success": False, "error": "File not found"}
        
        content = path.read_text().strip()
        lines = [line.strip() for line in content.splitlines() 
                if line.strip() and not line.startswith('#')]
        
        if not lines:
            cprint("No packages found in file", "WARNING")
            return {"success": False, "error": "No packages found"}
        
        cprint(f"Found {len(lines)} packages to install", "INFO")
        
        results = {
            "total_packages": len(lines),
            "successful": 0,
            "failed": 0,
            "results": []
        }
        
        progress = ProgressBar(len(lines), "Installing packages", "packages")
        
        for line in lines:
            # Parse package name (handle version specifiers)
            pkg_name = re.split(r'[=<>!]', line)[0].strip()
            
            cprint(f"Installing {pkg_name}...", "INFO")
            success, attempts = install_package(line)
            
            result = {
                "package": line,
                "success": success,
                "attempts": len(attempts)
            }
            results["results"].append(result)
            
            if success:
                results["successful"] += 1
            else:
                results["failed"] += 1
            
            progress.update(1)
        
        progress.finish()
        
        # Installation summary
        cprint(f"\nInstallation Summary:", "CYAN")
        cprint(f"  Successful: {results['successful']}/{results['total_packages']}", "SUCCESS")
        cprint(f"  Failed: {results['failed']}/{results['total_packages']}", "ERROR" if results['failed'] > 0 else "SUCCESS")
        
        return results
        
    except Exception as e:
        cprint(f"Error reading file: {e}", "ERROR")
        return {"success": False, "error": str(e)}

def export_packages(manager: str, output_file: Optional[str] = None) -> bool:
    """Exports installed packages to requirements file format"""
    cprint(f"Exporting packages from {_manager_human(manager)}...", "INFO")
    
    try:
        packages = package_db.get_installed_packages(manager)
        
        if not packages:
            cprint(f"No packages installed via {_manager_human(manager)} found in CrossFire database", "WARNING")
            return False
        
        # Generate requirements content
        lines = []
        for pkg in sorted(packages, key=lambda x: x['name']):
            if pkg['version'] and pkg['version'] != 'unknown':
                lines.append(f"{pkg['name']}=={pkg['version']}")
            else:
                lines.append(pkg['name'])
        
        content = '\n'.join(lines) + '\n'
        
        # Determine output file path
        if output_file:
            out_path = Path(output_file)
        else:
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            out_path = Path(f"crossfire_{manager}_requirements_{timestamp}.txt")
        
        # Write requirements file
        out_path.write_text(content)
        
        cprint(f"Exported {len(packages)} packages to: {out_path}", "SUCCESS")
        cprint(f"Install with: crossfire --install-from {out_path}", "INFO")
        
        return True
        
    except Exception as e:
        cprint(f"Export failed: {e}", "ERROR")
        return False
```

### Manager Installation System

```python
def install_manager(manager: str) -> bool:
    """Installs a package manager if supported on current OS"""
    manager = manager.lower()
    
    # Manager installation metadata
    MANAGER_SETUP = {
        "pip": {
            "os": ["windows", "linux", "macos"],
            "install": "Pip is bundled with Python 3.4+. Run: python -m ensurepip --upgrade",
            "install_cmd": [sys.executable, "-m", "ensurepip", "--upgrade"]
        },
        "npm": {
            "os": ["windows", "linux", "macos"],
            "install": "https://nodejs.org/ (download Node.js which includes npm)",
            "install_cmd": None  # Manual installation required
        },
        "apt": {
            "os": ["linux"],
            "install": "APT is preinstalled on Debian/Ubuntu systems",
            "install_cmd": None
        },
        "dnf": {
            "os": ["linux"],
            "install": "DNF is preinstalled on Fedora systems",
            "install_cmd": None
        },
        "yum": {
            "os": ["linux"],
            "install": "YUM is preinstalled on RHEL/CentOS systems",
            "install_cmd": None
        },
        "pacman": {
            "os": ["linux"],
            "install": "pacman is preinstalled on Arch Linux",
            "install_cmd": None
        },
        "zypper": {
            "os": ["linux"],
            "install": "zypper is preinstalled on openSUSE",
            "install_cmd": None
        },
        "apk": {
            "os": ["linux"],
            "install": "apk is bundled with Alpine Linux",
            "install_cmd": None
        },
        "brew": {
            "os": ["macos", "linux"],
            "install": '/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"',
            "install_cmd": None  # Requires shell execution
        },
        "choco": {
            "os": ["windows"],
            "install": 'Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString("https://chocolatey.org/install.ps1"))',
            "install_cmd": None  # Requires PowerShell
        },
        "winget": {
            "os": ["windows"],
            "install": "winget is preinstalled on Windows 11. Install via Microsoft Store on Windows 10.",
            "install_cmd": None
        },
        "snap": {
            "os": ["linux"],
            "install": "Install snapd package",
            "install_cmd": ["sudo", "apt", "install", "-y", "snapd"]
        },
        "flatpak": {
            "os": ["linux"],
            "install": "Install flatpak package",
            "install_cmd": ["sudo", "apt", "install", "-y", "flatpak"]
        },
    }
    
    info = MANAGER_SETUP.get(manager)
    if not info:
        cprint(f"Manager '{manager}' not supported.", "ERROR")
        return False
    
    os_name = _os_type()
    if os_name not in info["os"]:
        cprint(f"{manager} is not supported on this OS ({os_name}).", "ERROR")
        return False
    
    # Check if already installed
    if _detect_installed_managers().get(manager, False):
        cprint(f"{_manager_human(manager)} is already installed.", "SUCCESS")
        return True
    
    cmd = info.get("install_cmd")
    install_msg = info.get("install")
    
    if cmd:
        cprint(f"Installing {_manager_human(manager)}...", "INFO")
        result = run_command(cmd, timeout=900, show_progress=True)
        if result.ok:
            cprint(f"Successfully installed {_manager_human(manager)}", "SUCCESS")
            return True
        else:
            cprint(f"Failed to install {_manager_human(manager)}: {result.err}", "ERROR")
            return False
    else:
        cprint(f"Manual installation required for {_manager_human(manager)}:", "WARNING")
        cprint(f"  {install_msg}", "INFO")
        return False
```

---

## 17. Command Line Interface

### Argument Parser Configuration

```python
def create_parser() -> argparse.ArgumentParser:
    """Creates comprehensive command-line argument parser with detailed help"""
    parser = argparse.ArgumentParser(
        description="CrossFire — Universal Package Manager CLI",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Commands:
  General:
    --version                   Show CrossFire version
    -q, --quiet                 Quiet mode (errors only)
    -v, --verbose               Verbose output
    --json                      Output results in JSON format

  Package Management:
    --list-managers             List all supported package managers with status
    --install-manager <NAME>    Install a specific package manager (pip, npm, brew, etc.)
    --list-installed            Show packages installed via CrossFire
    -s, --search <QUERY>        Search across PyPI, NPM, and Homebrew
    -i, --install <PKG>         Install a package by name
    -r, --remove <PKG>          Remove/uninstall a package
    --manager <NAME>            Preferred manager to use (pip, npm, apt, brew, etc.)
    --install-from <FILE>       Install packages from a requirements file
    --export <MANAGER>          Export installed packages list
    -o, --output <FILE>         Output file for export command

  System Management:
    -um, --update-manager <NAME> Update specific manager or 'ALL'
    -cu, --crossupdate [URL]     Self-update from URL (default: GitHub)
    --sha256 <HASH>             Expected SHA256 hash for update verification
    --cleanup                   Clean package manager caches
    --health-check              Run comprehensive system health check
    --stats                     Show detailed package statistics
    --setup [DIR]               Install CrossFire launcher (optionally at a specific directory)

  Network Testing:
    --speed-test                Test internet download speed
    --ping-test                 Test network latency to various hosts
    --test-url <URL>            Custom URL for speed testing
    --test-duration <SECONDS>   Duration for speed test (default: 10s)

  Search Options:
    --search-limit <N>          Limit search results (default: 20)

Examples:
  crossfire --setup
  crossfire -s "web framework"
  crossfire -i requests --manager pip
  crossfire --install-manager brew
  crossfire --list-installed
  crossfire --install-from requirements.txt
  crossfire --export pip -o my_packages.txt
  crossfire --health-check
  crossfire --speed-test
        """,
    )

    # Version information
    parser.add_argument("--version", action="version", version=f"CrossFire {__version__}")

    # General options
    parser.add_argument("--json", action="store_true", help="Output results in JSON format")
    parser.add_argument("-q", "--quiet", action="store_true", help="Quiet mode (errors only)")
    parser.add_argument("-v", "--verbose", action="store_true", help="Verbose output")

    # Package management commands
    parser.add_argument("--list-managers", action="store_true", help="List all supported managers and their status")
    parser.add_argument("--install-manager", metavar="NAME", help="Install a specific package manager (pip, npm, brew, etc.)")
    parser.add_argument("--list-installed", action="store_true", help="Show packages installed via CrossFire")
    parser.add_argument("-s", "--search", metavar="QUERY", help="Search across real package repositories (PyPI, NPM, Homebrew)")
    parser.add_argument("-i", "--install", metavar="PKG", help="Install a package by name")
    parser.add_argument("-r", "--remove", metavar="PKG", help="Remove/uninstall a package")
    parser.add_argument("--manager", metavar="NAME", help="Preferred manager to use (pip, npm, apt, brew, etc.)")
    parser.add_argument("--install-from", metavar="FILE", help="Install packages from requirements file")
    parser.add_argument("--export", metavar="MANAGER", help="Export installed packages list")
    parser.add_argument("-o", "--output", metavar="FILE", help="Output file for export command")
    
    # System management commands
    parser.add_argument("-um", "--update-manager", metavar="NAME", help="Update specific manager or 'ALL'")
    parser.add_argument("-cu", "--crossupdate", nargs="?", const=DEFAULT_UPDATE_URL, metavar="URL",
                         help="Self-update from URL (default: GitHub)")
    parser.add_argument("--sha256", metavar="HASH", help="Expected SHA256 hash for update verification")
    parser.add_argument("--cleanup", action="store_true", help="Clean package manager caches")
    parser.add_argument("--health-check", action="store_true", help="Run comprehensive system health check")
    parser.add_argument("--stats", action="store_true", help="Show detailed package manager statistics")
    parser.add_argument("--setup", nargs="?", const="", metavar="DIR",
                        help="Install CrossFire launcher (optionally at a specific directory)")

    # Network testing commands
    parser.add_argument("--speed-test", action="store_true", help="Test internet download speed")
    parser.add_argument("--ping-test", action="store_true", help="Test network latency to various hosts")
    parser.add_argument("--test-url", metavar="URL", help="Custom URL for speed testing")
    parser.add_argument("--test-duration", type=int, default=10, metavar="SECONDS",
                         help="Duration for speed test (default: 10s)")

    # Search options
    parser.add_argument("--search-limit", type=int, default=20, metavar="N",
                         help="Limit search results (default: 20)")

    return parser
```

### Status Dashboard

```python
def show_enhanced_status() -> int:
    """Displays comprehensive status dashboard when no commands specified"""
    # Welcome header
    cprint("=" * 60, "CYAN")
    cprint(f"CrossFire v{__version__} — Universal Package Manager", "SUCCESS")
    cprint(f"System: {OS_NAME} {DISTRO_NAME} {DISTRO_VERSION} ({ARCH})", "INFO")
    cprint("=" * 60, "CYAN")
    
    status_info = list_managers_status()

    if LOG.json_mode:
        output = {
            "version": __version__,
            "os": OS_NAME,
            "distro": DISTRO_NAME,
            "distro_version": DISTRO_VERSION,
            "arch": ARCH,
            "managers": status_info,
            "crossfire_packages": len(package_db.get_installed_packages())
        }
        print(json.dumps(output, indent=2, ensure_ascii=False))
    else:
        installed_managers = sorted([m for m, s in status_info.items() if s == "Installed"])
        not_installed = sorted([m for m, s in status_info.items() if s != "Installed"])
        
        cprint(f"\nAvailable Package Managers ({len(installed_managers)}):", "SUCCESS")
        if installed_managers:
            for i, manager in enumerate(installed_managers, 1):
                cprint(f"  {i:2d}. {_manager_human(manager)}", "SUCCESS")
        else:
            cprint("      None found - consider installing pip, npm, brew, or apt", "WARNING")
        
        # Show CrossFire-managed packages
        crossfire_packages = package_db.get_installed_packages()
        cprint(f"\nCrossFire-Managed Packages: {len(crossfire_packages)}", "INFO")
        if crossfire_packages:
            recent = crossfire_packages[:3]  # Show 3 most recent
            for pkg in recent:
                cprint(f"  • {pkg['name']} via {_manager_human(pkg['manager'])}", "SUCCESS")
            if len(crossfire_packages) > 3:
                cprint(f"  ... and {len(crossfire_packages) - 3} more", "MUTED")
            cprint(f"  Use: crossfire --list-installed to see all", "INFO")
            
        if not_installed and LOG.verbose:
            cprint(f"\nUnavailable Managers:", "MUTED")
            for manager in not_installed[:5]: # Show only first 5
                status = status_info[manager]
                cprint(f"      ○ {manager} ({status})", "MUTED")
            if len(not_installed) > 5:
                cprint(f"      ... and {len(not_installed) - 5} more", "MUTED")
        
        cprint("\nQuick Start:", "CYAN")
        cprint("    crossfire --setup              # Install CrossFire globally", "INFO")
        cprint("    crossfire -s 'python library'  # Real search across repositories", "INFO") 
        cprint("    crossfire -i numpy             # Install with automatic tracking", "INFO")
        cprint("    crossfire --install-manager brew  # Install package managers", "INFO")
        cprint("    crossfire --list-installed     # Show managed packages", "INFO")
        cprint("    crossfire --health-check       # System diagnostics", "INFO")
        cprint("    crossfire --help               # Show all commands", "INFO")
        
        if installed_managers:
            cprint(f"\nSystem Ready! Found {len(installed_managers)} package managers.", "SUCCESS")
        else:
            cprint(f"\nSetup Needed - No package managers detected.", "WARNING")
    
    return 0

def list_managers_status() -> Dict[str, str]:
    """Returns status of all supported package managers"""
    installed = _detect_installed_managers()
    status = {}
    for manager in MANAGER_INSTALL_HANDLERS.keys():
        if manager in installed and installed[manager]:
            status[manager] = "Installed"
        else:
            status[manager] = "Not Installed"
    return status
```

---

## 18. Main Entry Point & Routing

### Command Router

```python
def main(argv: Optional[List[str]] = None) -> int:
    """Main execution entry point with comprehensive command routing"""
    parser = create_parser()
    args = parser.parse_args(argv)

    # Apply logging configuration
    LOG.quiet = args.quiet
    LOG.verbose = args.verbose
    LOG.json_mode = args.json
    
    try:
        # Setup command
        if args.setup is not None:
            cprint("Running production setup...", "INFO")
            
            path_success = add_to_path_safely()
            installed_path = install_launcher(args.setup if args.setup else None)
            
            if installed_path and path_success:
                cprint(f"\nSetup Complete!", "SUCCESS")
                cprint("    • CrossFire is now available globally as 'crossfire'", "SUCCESS")
                cprint("    • Restart your terminal or run: source ~/.bashrc", "INFO")
                cprint("    • Try: crossfire -s 'python library' to test search", "CYAN")
                cprint("    • Database initialized for package tracking", "INFO")
            else:
                cprint("Setup completed with some issues.", "WARNING")
            return 0

        # Manager installation
        if args.install_manager:
            manager = args.install_manager.lower()
            success = install_manager(manager)
            if LOG.json_mode:
                print(json.dumps({"manager": manager, "success": success}, indent=2))
            return 0 if success else 1

        # Network testing
        if args.speed_test:
            test_url = args.test_url
            duration = args.test_duration
            result = SpeedTest.test_download_speed(test_url, duration)
            if LOG.json_mode:
                print(json.dumps(result, indent=2))
            return 0 if result.get("ok") else 1
        
        if args.ping_test:
            result = SpeedTest.ping_test()
            if LOG.json_mode:
                print(json.dumps(result, indent=2))
            return 0

        # Update commands
        if args.crossupdate is not None:
            url = args.crossupdate or DEFAULT_UPDATE_URL
            success = cross_update(url, verify_sha256=args.sha256)
            return 0 if success else 1
        
        if args.update_manager:
            target = args.update_manager.upper()
            if target == "ALL":
                results = _update_all_managers()
            else:
                # Find proper case manager name
                proper_name = None
                for name in MANAGER_INSTALL_HANDLERS.keys():
                    if name.upper() == target:
                        proper_name = name
                        break
                
                if not proper_name:
                    cprint(f"Unknown package manager: {args.update_manager}", "ERROR")
                    return 1
                    
                name, ok, msg = _update_manager(proper_name)
                results = {name: {"ok": str(ok).lower(), "msg": msg}}
                cprint(f"{name}: {msg}", "SUCCESS" if ok else "ERROR")
                
            if LOG.json_mode:
                print(json.dumps(results, indent=2, ensure_ascii=False))
            return 0 if all(r.get("ok") == "true" for r in results.values()) else 1

        # Information commands
        if args.list_managers:
            status_info = list_managers_status()
            if LOG.json_mode:
                print(json.dumps(status_info, indent=2, ensure_ascii=False))
            else:
                cprint("Package Manager Status:", "INFO")
                for manager, status in sorted(status_info.items()):
                    if status == "Installed":
                        status_icon = f"Available"
                        color = "SUCCESS"
                    elif status == "Not Installed":
                        status_icon = f"Not Available"
                        color = "MUTED"
                    else:
                        status_icon = f"Unknown"
                        color = "WARNING"
                    cprint(f" {manager}: {status}", color)
                    
                cprint(f"\nInstall managers with: crossfire --install-manager <name>", "INFO")
            return 0
        
        if args.list_installed:
            show_installed_packages()
            return 0
        
        if args.stats:
            show_statistics()
            return 0
        
        if args.health_check:
            results = health_check()
            if LOG.json_mode:
                print(json.dumps(results, indent=2, default=str))
            return 0 if results["overall_status"] == "healthy" else 1

        # Search command
        if args.search:
            results = search_engine.search(args.search, args.manager, args.search_limit)
            
            if LOG.json_mode:
                output = {
                    "query": args.search, 
                    "results": [r.to_dict() for r in results],
                    "total_found": len(results)
                }
                print(json.dumps(output, indent=2, ensure_ascii=False))
            else:
                if not results:
                    cprint(f"No packages found for '{args.search}'", "WARNING")
                    cprint("Try different keywords or check internet connection", "INFO")
                    return 1
                
                cprint(f"Search Results for '{args.search}' (Found {len(results)})", "SUCCESS")
                cprint("=" * 70, "CYAN")
                
                for i, pkg in enumerate(results, 1):
                    # Relevance indicator
                    stars = min(5, max(1, int(pkg.relevance_score // 20)))
                    relevance_stars = "⭐" * stars
                    
                    cprint(f"\n{i:2d}. {pkg.name} ({_manager_human(pkg.manager)}) {relevance_stars}", "SUCCESS")
                    if pkg.version:
                        cprint(f"      Version: {pkg.version}", "INFO")
                    if pkg.description:
                        desc = pkg.description[:120] + "..." if len(pkg.description) > 120 else pkg.description
                        cprint(f"      {desc}", "MUTED")
                    if pkg.homepage:
                        cprint(f"      {pkg.homepage}", "CYAN")
                
                cprint(f"\nInstall with: crossfire -i <package_name>", "INFO")
                
            return 0
        
        # Package management commands
        if args.install:
            pkg = args.install
            success, attempts = install_package(pkg, preferred_manager=args.manager)
            
            if LOG.json_mode:
                output = {
                    "package": pkg, 
                    "success": success, 
                    "attempts": [
                        {
                            "manager": m, 
                            "result": {
                                "ok": r.ok, 
                                "code": r.code, 
                                "stdout": r.out, 
                                "stderr": r.err
                            }
                        } for m, r in attempts
                    ]
                }
                print(json.dumps(output, indent=2, ensure_ascii=False))
            return 0 if success else 1
        
        if args.remove:
            success, attempts = remove_package(args.remove, args.manager)
            
            if LOG.json_mode:
                output = {
                    "package": args.remove, 
                    "success": success, 
                    "attempts": [
                        {
                            "manager": m, 
                            "result": {
                                "ok": r.ok, 
                                "code": r.code, 
                                "stdout": r.out, 
                                "stderr": r.err
                            }
                        } for m, r in attempts
                    ]
                }
                print(json.dumps(output, indent=2, ensure_ascii=False))
            return 0 if success else 1
        
        if args.install_from:
            results = bulk_install_from_file(args.install_from)
            if LOG.json_mode:
                print(json.dumps(results, indent=2, default=str))
            return 0 if results.get("success", False) else 1
        
        if args.export:
            success = export_packages(args.export, args.output)
            return 0 if success else 1
        
        if args.cleanup:
            results = cleanup_system()
            if LOG.json_mode:
                print(json.dumps(results, indent=2, ensure_ascii=False))
            return 0 if any(r.get("ok") == "true" for r in results.values()) else 1
        
        # No specific command - show status dashboard
        return show_enhanced_status()
        
    except KeyboardInterrupt:
        cprint("\nOperation cancelled by user.", "WARNING")
        return 1
    except Exception as e:
        cprint(f"Unexpected error: {e}", "ERROR")
        if LOG.verbose:
            import traceback
            traceback.print_exc()
        return 1
```

### Entry Point & Dependency Check

```python
if __name__ == "__main__":
    # Ensure required dependencies are available
    try:
        import requests
    except ImportError:
        print("Missing required dependency 'requests'. Install with: pip install requests")
        sys.exit(1)
    
    sys.exit(main())
```

**Command Routing Logic:**
- Arguments processed in specific order (setup first, then updates, then info, then actions)
- Each command returns appropriate exit code (0 for success, 1 for failure)
- JSON mode vs console mode handled per command
- Keyboard interrupt (Ctrl+C) handled gracefully
- Unexpected exceptions logged with optional traceback in verbose mode
- Missing requests dependency checked at startup

---

## 19. Error Handling & Edge Cases

### Known Error Scenarios

1. **Network Issues**
   - **Search failures**: Individual manager searches fail silently, partial results returned
   - **Download timeouts**: Self-update and speed tests handle timeouts with fallback
   - **DNS resolution**: No specific handling, relies on system resolver

2. **Permission Issues**
   - **Sudo requirements**: System manager commands fail if user lacks sudo privileges
   - **File permissions**: Self-update fails if script location is read-only
   - **PATH modification**: Shell config updates may fail due to file permissions

3. **Package Manager Issues**
   - **Interactive prompts**: Commands requiring user input will hang indefinitely
   - **Package conflicts**: Dependency conflicts handled by underlying managers, not CrossFire
   - **Version mismatches**: CrossFire attempts requested versions but managers may install different ones

4. **Database Issues**
   - **Corruption**: Database corruption detected during health checks but no auto-repair
   - **Concurrency**: No locking mechanism for concurrent CrossFire instances
   - **Schema changes**: No migration system for future schema updates

5. **Platform-Specific Issues**
   - **Windows PATH**: Setup doesn't automatically update system PATH on Windows
   - **Shell variations**: PATH updates may not work with non-standard shells
   - **Package manager differences**: Command formats vary between Linux distributions

### Error Recovery Mechanisms

```python
# Example error handling patterns used throughout:

# 1. Graceful degradation in search
try:
    results = self._search_pypi(query)
    all_results.extend(results)
except Exception as e:
    cprint(f"PyPI search failed: {e}", "WARNING")
    # Continue with other managers

# 2. Retry logic in command execution
for attempt in range(retries + 1):
    try:
        result = subprocess.run(...)
        if result.returncode == 0:
            break
    except Exception as e:
        if attempt == retries:
            return RunResult(False, -1, "", str(e))

# 3. Fallback behavior in detection
try:
    import distro
    DISTRO_NAME = distro.id()
except ImportError:
    # Platform-specific fallback
    if OS_NAME == "Darwin":
        DISTRO_NAME = "macOS"
```

---

## 20. Security Considerations

### Network Security
- **HTTPS enforcement**: Self-update uses HTTPS but no certificate pinning
- **SHA256 verification**: Optional but not enforced by default
- **User-Agent disclosure**: Identifies requests as coming from CrossFire
- **URL validation**: No validation of user-provided URLs for downloads

### Privilege Escalation
- **Sudo usage**: Hardcoded sudo for system package managers
- **Command injection**: Package names passed directly to shell commands
- **Path traversal**: No validation of file paths in import/export operations

### Data Privacy
- **Command logging**: Full install commands stored in database including arguments
- **Network tracking**: Search queries sent to public APIs (PyPI, NPM, etc.)
- **Local data**: Package database contains installation history

### Code Execution Risks
- **Self-update mechanism**: Downloads and executes Python code from remote URL
- **Package managers**: All underlying managers can execute arbitrary post-install scripts
- **Shell command execution**: Some commands use shell=True with potential injection risks

**Risk Mitigation Recommendations:**
1. Implement URL allowlist for self-updates
2. Add package name sanitization for shell commands
3. Enable SHA256 verification by default
4. Add option to disable command logging
5. Implement certificate pinning for critical URLs

---

## 21. Known Bugs & Limitations

### Critical Bugs

1. **Brew Update Command (Line ~1300)**
   ```python
   "brew": ["brew", "update", "&&", "brew", "upgrade"]  # BUG!
   ```
   The `&&` operator is passed as a literal argument instead of shell operator.
   **Fix**: Use `"brew update && brew upgrade"` with `shell=True` or split into two commands.

2. **PyPI Search Limitation**
   - Only exact package name matches work
   - No full-text search capability
   - Many relevant packages won't be found

3. **Windows PATH Setup**
   - Installer copies launcher but doesn't update system PATH
   - Users must manually configure PATH or file associations

### Design Limitations

1. **Interactive Command Support**
   - Commands requiring TTY input (license agreements) will hang
   - No expect/pexpect integration for automated responses
   - Progress indicators interfere with interactive prompts

2. **Concurrency Issues**
   - No database locking for multiple CrossFire instances
   - Race conditions possible during parallel installations
   - Search concurrency limited to prevent API rate limiting

3. **Platform Inconsistencies**
   - Package manager command formats vary by distribution
   - Heuristic-based manager selection can be wrong
   - Version extraction regex varies by manager output format

4. **Error Reporting**
   - Generic error messages for complex failure scenarios
   - No detailed troubleshooting guidance
   - Limited context for permission-related failures

### Performance Issues

1. **Cold Start Delays**
   - First Homebrew search downloads ~5MB JSON file
   - Initial manager detection tests all possible commands
   - Database schema creation on first run

2. **Memory Usage**
   - Homebrew formula list cached in memory
   - Large package lists from CLI searches loaded entirely
   - No streaming for bulk operations

3. **Network Dependencies**
   - All search operations require internet connectivity
   - No offline mode for previously searched packages
   - Timeout handling may be too aggressive for slow connections

---

## 22. Extension Guide

### Adding a New Package Manager

To add support for a new package manager named "foo":

#### Step 1: Command Builders
```python
def _foo_install(pkg: str) -> List[str]:
    """Build install command for foo package manager"""
    return ["foo", "install", pkg]

def _foo_remove(pkg: str) -> List[str]:
    """Build remove command for foo package manager"""  
    return ["foo", "uninstall", pkg]
```

#### Step 2: Register Commands
```python
MANAGER_INSTALL_HANDLERS["foo"] = _foo_install
MANAGER_REMOVE_HANDLERS["foo"] = _foo_remove
```

#### Step 3: Detection Logic
Detection is automatic via `shutil.which("foo")`. For special cases:
```python
def _detect_installed_managers():
    # ... existing code ...
    
    # Special case for foo
    if name == "foo":
        available[name] = custom_foo_detection_logic()
    else:
        available[name] = shutil.which(name) is not None
```

#### Step 4: Optional Enhancements

**Search Integration:**
```python
def _search_foo(self, query: str) -> List[SearchResult]:
    """Search foo repository"""
    # Implement API or CLI-based search
    return results

# Add to RealSearchEngine.search():
manager_funcs["foo"] = self._search_foo
```

**Cleanup Support:**
```python
# Add to cleanup_system():
cleanup_commands["foo"] = ["foo", "clean", "--all"]
```

**Update Support:**
```python
# Add to _update_manager():
update_commands["foo"] = ["foo", "update"]
```

**Package Type Heuristics:**
```python
def _looks_like_foo_pkg(pkg: str) -> bool:
    """Detect foo-specific package patterns"""
    return pkg.startswith("foo-") or "foo.yaml" in pkg

# Use in _ordered_install_manager_candidates()
```

#### Step 5: Manager Installation
```python
# Add to install_manager():
MANAGER_SETUP["foo"] = {
    "os": ["linux", "macos"],
    "install": "Install foo from https://foo.example.com",
    "install_cmd": ["curl", "-sSL", "https://foo.example.com/install.sh", "|", "bash"]
}
```

### Adding New CLI Commands

#### Example: Adding Package Information Command

```python
# Add to create_parser():
parser.add_argument("--info", metavar="PKG", help="Show detailed package information")

# Add to main() routing:
if args.info:
    info = get_package_info(args.info, args.manager)
    if LOG.json_mode:
        print(json.dumps(info, indent=2))
    else:
        display_package_info(info)
    return 0 if info else 1

def get_package_info(pkg: str, manager: str = None) -> Dict[str, Any]:
    """Get detailed information about a package"""
    # Implementation here
    pass

def display_package_info(info: Dict[str, Any]):
    """Display package info in human-readable format"""
    # Implementation here  
    pass
```

### Customizing Search Behavior

#### Adding Custom Relevance Scoring
```python
def custom_relevance_score(pkg_name: str, description: str, query: str) -> float:
    """Custom relevance scoring algorithm"""
    score = 0
    query_lower = query.lower()
    
    # Exact name match
    if query_lower == pkg_name.lower():
        score += 100
    
    # Name contains query
    if query_lower in pkg_name.lower():
        score += 80
    
    # Description relevance
    desc_words = description.lower().split()
    query_words = query_lower.split()
    
    for q_word in query_words:
        if q_word in desc_words:
            score += 20
    
    return score

# Use in search result creation:
result.relevance_score = custom_relevance_score(name, desc, query)
```

---

## 23. Complete Command Reference

### General Commands
```bash
crossfire --version              # Show version: "CrossFire 1.0 - BlackBase"
crossfire --help                 # Show full help text with examples
crossfire -q                     # Quiet mode (errors only)
crossfire -v                     # Verbose mode (debug output)
crossfire --json                 # JSON output for all commands
```

### Package Manager Commands
```bash
crossfire --list-managers        # Show all 13 supported managers + status
crossfire --install-manager pip  # Install pip (auto-detects Python)
crossfire --install-manager brew # Install Homebrew (macOS/Linux)
crossfire --install-manager npm  # Shows Node.js download instructions
```

### Search & Discovery
```bash
crossfire -s "web framework"     # Real search across PyPI, NPM, Homebrew
crossfire -s numpy --manager pip # Limit search to specific manager
crossfire -s react --search-limit 50  # Show up to 50 results
```

### Package Installation
```bash
crossfire -i requests            # Install with automatic manager selection
crossfire -i express --manager npm    # Force specific manager
crossfire --install-from requirements.txt  # Bulk install from file
```

### Package Removal
```bash
crossfire -r requests            # Remove with automatic manager detection
crossfire -r express --manager npm     # Force specific manager
```

### Package Tracking
```bash
crossfire --list-installed       # Show packages installed via CrossFire
crossfire --stats                # Detailed statistics with charts
crossfire --export pip           # Export pip packages to file
crossfire --export npm -o packages.txt  # Export to specific file
```

### System Management
```bash
crossfire --setup                # Install launcher globally + PATH
crossfire --setup ~/bin          # Install launcher to custom directory
crossfire --cleanup              # Clean all package manager caches
crossfire --health-check         # Comprehensive system diagnostics
```

### Manager Updates
```bash
crossfire -um pip                # Update pip to latest version
crossfire -um ALL                # Update all available managers
```

### Self-Update
```bash
crossfire -cu                    # Update from default GitHub URL
crossfire -cu https://example.com/crossfire.py  # Custom URL
crossfire -cu --sha256 abc123... # Verify download with hash
```

### Network Testing  
```bash
crossfire --speed-test           # Test download speed (10s default)
crossfire --speed-test --test-duration 30  # Custom duration
crossfire --speed-test --test-url http://example.com/file  # Custom URL
crossfire --ping-test            # Test latency to common hosts
```

### JSON API Mode
```bash
crossfire --json -s numpy        # Machine-readable search results
crossfire --json --health-check  # JSON health report
crossfire --json --list-installed # JSON package list
crossfire --json -i requests     # JSON installation report
```

### Exit Codes
- `0`: Success or healthy status
- `1`: Failure, error, or unhealthy status

### Configuration Files
- **Database**: `~/.crossfire/packages.db` (SQLite)
- **Cache**: `~/.crossfire/cache/` (downloads, API responses)
- **Launcher**: `~/.local/bin/crossfire` (Unix) or `%LOCALAPPDATA%\CrossFire\crossfire.exe` (Windows)
