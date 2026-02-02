# File Management: Organizing Your Agent's Digital Workspace

Files are the backbone of agent persistence. From logs to outputs, configs to caches - how you manage files determines whether your agent is a well-oiled machine or a chaotic mess.

## File System Philosophy

**Core Principles:**
1. **Predictable paths** - Know where everything lives
2. **Clean separation** - Temporary vs persistent, user vs system
3. **Safe operations** - Never lose data, always recoverable
4. **Efficient access** - Fast reads, atomic writes

## Strategy 1: Structured Workspace

```python
import os
from pathlib import Path
from datetime import datetime
from typing import Optional
import shutil

class AgentWorkspace:
    """Manage agent file system with clear structure"""
    
    def __init__(self, base_path: str = "~/.agent"):
        self.base = Path(base_path).expanduser()
        self._init_structure()
    
    def _init_structure(self):
        """Create standard directory structure"""
        directories = [
            "workspace",        # Active working files
            "workspace/temp",   # Temporary files (auto-cleaned)
            "data",            # Persistent data
            "data/cache",      # Cached API responses, etc.
            "data/memory",     # Agent memory/state
            "config",          # Configuration files
            "logs",            # Log files
            "logs/archive",    # Archived old logs
            "outputs",         # Generated outputs
            "outputs/reports", # Reports and exports
            "inputs",          # User-provided inputs
            "backups",         # Backup files
        ]
        
        for dir_path in directories:
            (self.base / dir_path).mkdir(parents=True, exist_ok=True)
    
    @property
    def workspace(self) -> Path:
        return self.base / "workspace"
    
    @property
    def temp(self) -> Path:
        return self.base / "workspace" / "temp"
    
    @property
    def data(self) -> Path:
        return self.base / "data"
    
    @property
    def cache(self) -> Path:
        return self.base / "data" / "cache"
    
    @property
    def memory(self) -> Path:
        return self.base / "data" / "memory"
    
    @property
    def config(self) -> Path:
        return self.base / "config"
    
    @property
    def logs(self) -> Path:
        return self.base / "logs"
    
    @property
    def outputs(self) -> Path:
        return self.base / "outputs"
    
    def get_dated_path(self, base_dir: str, extension: str = "") -> Path:
        """Get path with current date for organizing files"""
        date_str = datetime.now().strftime("%Y-%m-%d")
        dir_path = self.base / base_dir / date_str
        dir_path.mkdir(parents=True, exist_ok=True)
        
        if extension:
            timestamp = datetime.now().strftime("%H%M%S")
            return dir_path / f"{timestamp}{extension}"
        return dir_path
    
    def cleanup_temp(self, max_age_hours: int = 24):
        """Remove old temporary files"""
        import time
        now = time.time()
        
        for file_path in self.temp.rglob("*"):
            if file_path.is_file():
                age_hours = (now - file_path.stat().st_mtime) / 3600
                if age_hours > max_age_hours:
                    file_path.unlink()
                    print(f"Cleaned up: {file_path}")
    
    def get_disk_usage(self) -> dict:
        """Get disk usage statistics"""
        usage = {}
        for dir_name in ["workspace", "data", "logs", "outputs"]:
            dir_path = self.base / dir_name
            total_size = sum(
                f.stat().st_size for f in dir_path.rglob("*") if f.is_file()
            )
            usage[dir_name] = {
                "size_mb": total_size / (1024 * 1024),
                "file_count": sum(1 for _ in dir_path.rglob("*") if _.is_file())
            }
        return usage

# Usage
workspace = AgentWorkspace()

# Get paths
config_file = workspace.config / "settings.json"
log_file = workspace.get_dated_path("logs", ".log")
output_file = workspace.outputs / "report.pdf"

# Periodic cleanup
workspace.cleanup_temp(max_age_hours=12)
```

## Strategy 2: Safe File Operations

```python
import json
import tempfile
import hashlib
from pathlib import Path
from typing import Any, Optional
from contextlib import contextmanager
import fcntl

class SafeFileOps:
    """Safe file operations that never lose data"""
    
    @staticmethod
    def atomic_write(path: Path, content: str | bytes, mode: str = "w"):
        """Write atomically using temp file + rename"""
        path = Path(path)
        path.parent.mkdir(parents=True, exist_ok=True)
        
        # Write to temp file in same directory
        temp_path = path.with_suffix(path.suffix + ".tmp")
        
        try:
            with open(temp_path, mode) as f:
                f.write(content)
                f.flush()
                os.fsync(f.fileno())  # Ensure written to disk
            
            # Atomic rename
            temp_path.rename(path)
            
        except Exception:
            # Clean up temp file on failure
            if temp_path.exists():
                temp_path.unlink()
            raise
    
    @staticmethod
    def atomic_json_write(path: Path, data: Any, indent: int = 2):
        """Write JSON atomically"""
        content = json.dumps(data, indent=indent, default=str)
        SafeFileOps.atomic_write(path, content)
    
    @staticmethod
    def safe_read(path: Path, default: Any = None) -> Any:
        """Read file with default on failure"""
        try:
            with open(path, "r") as f:
                return f.read()
        except (FileNotFoundError, PermissionError):
            return default
    
    @staticmethod
    def safe_json_read(path: Path, default: Any = None) -> Any:
        """Read JSON with default on failure"""
        try:
            with open(path, "r") as f:
                return json.load(f)
        except (FileNotFoundError, json.JSONDecodeError, PermissionError):
            return default if default is not None else {}
    
    @staticmethod
    @contextmanager
    def file_lock(path: Path, exclusive: bool = True):
        """Context manager for file locking"""
        lock_path = Path(str(path) + ".lock")
        lock_file = open(lock_path, "w")
        
        try:
            fcntl.flock(
                lock_file.fileno(),
                fcntl.LOCK_EX if exclusive else fcntl.LOCK_SH
            )
            yield
        finally:
            fcntl.flock(lock_file.fileno(), fcntl.LOCK_UN)
            lock_file.close()
    
    @staticmethod
    def backup_before_modify(path: Path, backup_dir: Path = None) -> Optional[Path]:
        """Create backup before modifying file"""
        if not path.exists():
            return None
        
        backup_dir = backup_dir or path.parent / "backups"
        backup_dir.mkdir(parents=True, exist_ok=True)
        
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        backup_path = backup_dir / f"{path.stem}_{timestamp}{path.suffix}"
        
        shutil.copy2(path, backup_path)
        return backup_path
    
    @staticmethod
    def get_file_hash(path: Path) -> str:
        """Get SHA256 hash of file"""
        sha256 = hashlib.sha256()
        with open(path, "rb") as f:
            for chunk in iter(lambda: f.read(8192), b""):
                sha256.update(chunk)
        return sha256.hexdigest()
    
    @staticmethod
    def safe_delete(path: Path, trash_dir: Path = None) -> bool:
        """Move to trash instead of permanent delete"""
        if not path.exists():
            return False
        
        trash_dir = trash_dir or Path("~/.agent/trash").expanduser()
        trash_dir.mkdir(parents=True, exist_ok=True)
        
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        trash_path = trash_dir / f"{path.name}_{timestamp}"
        
        shutil.move(str(path), str(trash_path))
        return True

# Usage
ops = SafeFileOps()

# Atomic write (never corrupts file)
ops.atomic_json_write(Path("config.json"), {"key": "value"})

# Read with default
config = ops.safe_json_read(Path("config.json"), default={"key": "default"})

# Locked write (prevents race conditions)
with ops.file_lock(Path("shared_state.json")):
    state = ops.safe_json_read(Path("shared_state.json"))
    state["updated"] = datetime.now().isoformat()
    ops.atomic_json_write(Path("shared_state.json"), state)

# Safe delete (recoverable)
ops.safe_delete(Path("old_file.txt"))
```

## Strategy 3: File-Based State Management

```python
import json
import time
from pathlib import Path
from typing import Dict, Any, Optional
from dataclasses import dataclass, asdict
from threading import Lock

@dataclass
class StateMetadata:
    created_at: str
    updated_at: str
    version: int
    checksum: str

class PersistentState:
    """File-backed state with auto-save and versioning"""
    
    def __init__(self, path: Path, auto_save: bool = True, save_interval: int = 30):
        self.path = Path(path)
        self.auto_save = auto_save
        self.save_interval = save_interval
        self._data: Dict[str, Any] = {}
        self._metadata: Optional[StateMetadata] = None
        self._dirty = False
        self._last_save = 0
        self._lock = Lock()
        
        self._load()
    
    def _load(self):
        """Load state from file"""
        if self.path.exists():
            try:
                with open(self.path, "r") as f:
                    stored = json.load(f)
                    self._data = stored.get("data", {})
                    if "metadata" in stored:
                        self._metadata = StateMetadata(**stored["metadata"])
            except (json.JSONDecodeError, KeyError):
                self._data = {}
                self._metadata = None
    
    def _save(self, force: bool = False):
        """Save state to file"""
        with self._lock:
            if not self._dirty and not force:
                return
            
            # Update metadata
            now = datetime.now().isoformat()
            checksum = hashlib.md5(json.dumps(self._data).encode()).hexdigest()
            
            if self._metadata:
                self._metadata.updated_at = now
                self._metadata.version += 1
                self._metadata.checksum = checksum
            else:
                self._metadata = StateMetadata(
                    created_at=now,
                    updated_at=now,
                    version=1,
                    checksum=checksum
                )
            
            stored = {
                "data": self._data,
                "metadata": asdict(self._metadata)
            }
            
            SafeFileOps.atomic_json_write(self.path, stored)
            self._dirty = False
            self._last_save = time.time()
    
    def _maybe_auto_save(self):
        """Auto-save if enabled and interval passed"""
        if self.auto_save and self._dirty:
            if time.time() - self._last_save > self.save_interval:
                self._save()
    
    def get(self, key: str, default: Any = None) -> Any:
        """Get value by key"""
        return self._data.get(key, default)
    
    def set(self, key: str, value: Any):
        """Set value by key"""
        with self._lock:
            self._data[key] = value
            self._dirty = True
        self._maybe_auto_save()
    
    def update(self, data: Dict[str, Any]):
        """Update multiple values"""
        with self._lock:
            self._data.update(data)
            self._dirty = True
        self._maybe_auto_save()
    
    def delete(self, key: str):
        """Delete key"""
        with self._lock:
            if key in self._data:
                del self._data[key]
                self._dirty = True
        self._maybe_auto_save()
    
    def all(self) -> Dict[str, Any]:
        """Get all data"""
        return self._data.copy()
    
    def flush(self):
        """Force save"""
        self._save(force=True)
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        self.flush()

# Usage
state = PersistentState(Path("~/.agent/data/state.json").expanduser())

# Read/write operations
state.set("last_run", datetime.now().isoformat())
state.set("task_count", state.get("task_count", 0) + 1)

# Batch update
state.update({
    "status": "running",
    "current_task": "process_data"
})

# Force save
state.flush()

# Auto-cleanup with context manager
with PersistentState(Path("session_state.json")) as session:
    session.set("started", datetime.now().isoformat())
    # ... do work ...
    session.set("completed", datetime.now().isoformat())
# Auto-saves on exit
```

## Strategy 4: Log Management

```python
import logging
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler
from pathlib import Path
from datetime import datetime
import gzip
import shutil

class LogManager:
    """Comprehensive logging setup for agents"""
    
    def __init__(self, log_dir: Path, app_name: str = "agent"):
        self.log_dir = Path(log_dir)
        self.log_dir.mkdir(parents=True, exist_ok=True)
        self.app_name = app_name
        self.loggers: Dict[str, logging.Logger] = {}
    
    def get_logger(
        self,
        name: str,
        level: int = logging.INFO,
        max_bytes: int = 10 * 1024 * 1024,  # 10MB
        backup_count: int = 5
    ) -> logging.Logger:
        """Get or create a logger with file rotation"""
        
        if name in self.loggers:
            return self.loggers[name]
        
        logger = logging.getLogger(f"{self.app_name}.{name}")
        logger.setLevel(level)
        
        # Prevent duplicate handlers
        if logger.handlers:
            return logger
        
        # File handler with rotation
        log_file = self.log_dir / f"{name}.log"
        file_handler = RotatingFileHandler(
            log_file,
            maxBytes=max_bytes,
            backupCount=backup_count
        )
        file_handler.setLevel(level)
        
        # Console handler
        console_handler = logging.StreamHandler()
        console_handler