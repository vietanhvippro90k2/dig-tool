# DigTool - Platform Compatibility Report

## Current Platform Dependencies

### Windows-Only Components

DigTool is currently **Windows-exclusive** due to extensive use of Windows-specific APIs. Here's a breakdown of the platform dependencies:

### 1. **Screen Capture System**
**Location**: `utils/screen_capture.py`, `utils/system_utils.py`

**Windows Dependencies:**
```python
import win32gui, win32ui, win32con, win32api
import ctypes.windll
```

**Specific APIs Used:**
- `win32gui.GetDesktopWindow()` - Desktop window handle
- `win32gui.GetWindowDC()` - Device context for screen capture
- `win32ui.CreateDCFromHandle()` - Compatible device context creation
- `win32ui.CreateBitmap()` - Bitmap creation for screen data
- `ctypes.windll.user32` - Direct Windows user32.dll access
- `ctypes.windll.gdi32` - Graphics device interface

### 2. **Input Simulation**
**Location**: `utils/system_utils.py`, `main.py`

**Windows Dependencies:**
```python
# Mouse clicking
win32api.mouse_event(win32con.MOUSEEVENTF_LEFTDOWN, 0, 0, 0, 0)
win32api.mouse_event(win32con.MOUSEEVENTF_LEFTUP, 0, 0, 0, 0)

# Cursor positioning
ctypes.windll.user32.SetCursorPos(*current_pos)

# Low-level input simulation
ctypes.windll.user32.SendInput(2, inputs, ctypes.sizeof(INPUT))
```

### 3. **DPI and Display Management**
**Location**: `main.py`, `utils/system_utils.py`

**Windows Dependencies:**
```python
# DPI awareness
ctypes.windll.shcore.SetProcessDpiAwareness(PROCESS_PER_MONITOR_DPI_AWARE)
ctypes.windll.user32.SetProcessDPIAware()

# Display metrics
ctypes.windll.user32.GetSystemMetrics(0)  # Screen width
ctypes.windll.gdi32.GetDeviceCaps(hdc, 88)  # DPI information
```

### 4. **Window Management**
**Location**: `utils/system_utils.py`, `interface/components.py`

**Windows Dependencies:**
```python
# Window enumeration and manipulation
win32gui.EnumWindows()
win32gui.SetForegroundWindow()
win32gui.GetWindowText()
win32gui.GetWindowRect()
```

## Cross-Platform Adaptation Strategy

### Phase 1: Abstract Platform-Specific Operations

#### 1. **Create Platform Abstraction Layer**

```python
# utils/platform_adapter.py
from abc import ABC, abstractmethod
import platform

class PlatformAdapter(ABC):
    @abstractmethod
    def capture_screen(self, bbox=None):
        pass
    
    @abstractmethod
    def send_click(self, x=None, y=None):
        pass
    
    @abstractmethod
    def set_cursor_position(self, x, y):
        pass
    
    @abstractmethod
    def get_screen_resolution(self):
        pass

class WindowsPlatformAdapter(PlatformAdapter):
    # Existing Windows implementation
    pass

class LinuxPlatformAdapter(PlatformAdapter):
    # New Linux implementation using X11/Wayland
    pass

class MacOSPlatformAdapter(PlatformAdapter):
    # New macOS implementation using Quartz
    pass

def get_platform_adapter():
    system = platform.system()
    if system == "Windows":
        return WindowsPlatformAdapter()
    elif system == "Linux":
        return LinuxPlatformAdapter()
    elif system == "Darwin":
        return MacOSPlatformAdapter()
    else:
        raise UnsupportedPlatformError(f"Platform {system} not supported")
```

#### 2. **Linux Implementation Options**

**Option A: PyAutoGUI (Simplest)**
```python
import pyautogui
import numpy as np
from PIL import ImageGrab

class LinuxPlatformAdapter(PlatformAdapter):
    def capture_screen(self, bbox=None):
        if bbox:
            screenshot = ImageGrab.grab(bbox)
        else:
            screenshot = pyautogui.screenshot()
        return np.array(screenshot)
    
    def send_click(self, x=None, y=None):
        if x is not None and y is not None:
            pyautogui.click(x, y)
        else:
            pyautogui.click()
    
    def set_cursor_position(self, x, y):
        pyautogui.moveTo(x, y)
    
    def get_screen_resolution(self):
        return pyautogui.size()
```

**Option B: X11 Direct (Better Performance)**
```python
import Xlib.display
import Xlib.X
import Xlib.protocol.event
from PIL import Image

class LinuxX11Adapter(PlatformAdapter):
    def __init__(self):
        self.display = Xlib.display.Display()
        self.root = self.display.screen().root
    
    def capture_screen(self, bbox=None):
        if bbox:
            x, y, width, height = bbox[0], bbox[1], bbox[2]-bbox[0], bbox[3]-bbox[1]
            raw = self.root.get_image(x, y, width, height, Xlib.X.ZPixmap, 0xffffffff)
        else:
            geometry = self.root.get_geometry()
            raw = self.root.get_image(0, 0, geometry.width, geometry.height, Xlib.X.ZPixmap, 0xffffffff)
        
        image = Image.frombytes("RGB", (raw.width, raw.height), raw.data, "raw", "BGRX")
        return np.array(image)
```

**Option C: Wayland Support**
```python
# For modern Linux distributions using Wayland
import subprocess
import tempfile
import os
from PIL import Image

class LinuxWaylandAdapter(PlatformAdapter):
    def capture_screen(self, bbox=None):
        # Use grim for Wayland screenshot
        with tempfile.NamedTemporaryFile(suffix='.png', delete=False) as tmp:
            if bbox:
                x, y, w, h = bbox[0], bbox[1], bbox[2]-bbox[0], bbox[3]-bbox[1]
                subprocess.run(['grim', '-g', f'{x},{y} {w}x{h}', tmp.name])
            else:
                subprocess.run(['grim', tmp.name])
            
            image = Image.open(tmp.name)
            os.unlink(tmp.name)
            return np.array(image)
```

### Phase 2: Dependency Management

#### Update Requirements
```python
# requirements_base.txt (cross-platform)
opencv-python
numpy
pillow
keyboard  # Works cross-platform
requests

# requirements_windows.txt
-r requirements_base.txt
pywin32
pyautoit

# requirements_linux.txt
-r requirements_base.txt
python-xlib
pynput
pyautogui

# requirements_macos.txt
-r requirements_base.txt
pyobjc-framework-Quartz
pynput
pyautogui
```

#### Dynamic Import Pattern
```python
# utils/platform_imports.py
import platform

if platform.system() == "Windows":
    try:
        import win32gui, win32ui, win32con, win32api
        import ctypes.windll
        PLATFORM_AVAILABLE = True
    except ImportError:
        PLATFORM_AVAILABLE = False
elif platform.system() == "Linux":
    try:
        import Xlib.display
        import pyautogui
        PLATFORM_AVAILABLE = True
    except ImportError:
        PLATFORM_AVAILABLE = False
else:
    PLATFORM_AVAILABLE = False
```

### Phase 3: Configuration Management

#### Platform-Specific Settings
```python
# config/platform_config.py
import platform

PLATFORM_CONFIGS = {
    "Windows": {
        "screen_capture_method": "win32",
        "input_method": "win32",
        "dpi_aware": True,
        "max_fps": 120
    },
    "Linux": {
        "screen_capture_method": "x11",  # or "wayland"
        "input_method": "xlib",
        "dpi_aware": False,  # Handled differently on Linux
        "max_fps": 60  # More conservative for X11
    },
    "Darwin": {  # macOS
        "screen_capture_method": "quartz",
        "input_method": "quartz",
        "dpi_aware": True,
        "max_fps": 60
    }
}

def get_platform_config():
    return PLATFORM_CONFIGS.get(platform.system(), {})
```

## Implementation Priority

### High Priority (Essential for Cross-Platform)
1. ✅ **Screen Capture Abstraction** - Core functionality
2. ✅ **Input Simulation Abstraction** - Essential for automation
3. ✅ **Platform Detection** - Runtime adaptation

### Medium Priority (Enhanced Compatibility)
1. 🔶 **Display Management** - DPI handling per platform
2. 🔶 **Window Management** - Platform-specific window APIs
3. 🔶 **Performance Optimization** - Platform-specific optimizations

### Low Priority (Nice to Have)
1. 🔹 **Native Look and Feel** - Platform-specific UI themes
2. 🔹 **Platform-Specific Features** - Advanced integrations
3. 🔹 **Distribution Packaging** - Platform-specific installers

## Testing Strategy

### Platform Testing Matrix
| Feature | Windows | Linux (X11) | Linux (Wayland) | macOS |
|---------|---------|-------------|-----------------|-------|
| Screen Capture | ✅ Win32 | 🔶 PyAutoGUI | 🔶 Grim | 🔶 Quartz |
| Input Simulation | ✅ Win32 | 🔶 PyAutoGUI | 🔶 wtype | 🔶 Quartz |
| Window Management | ✅ Win32 | ❌ Limited | ❌ Limited | 🔶 Quartz |
| DPI Awareness | ✅ Native | 🔶 Manual | 🔶 Manual | 🔶 Retina |

### Test Environment Setup
```bash
# Linux test environment
sudo apt-get install python3-tk python3-dev
pip install python-xlib pyautogui pynput

# Wayland testing
sudo apt-get install grim wtype

# Virtual display for headless testing
sudo apt-get install xvfb
export DISPLAY=:99
Xvfb :99 -screen 0 1024x768x24 > /dev/null 2>&1 &
```

## Migration Risks and Mitigation

### High-Risk Areas
1. **Performance Degradation** - PyAutoGUI is slower than native Win32
   - **Mitigation**: Profile different capture methods, implement caching
2. **Accuracy Loss** - Different screen capture methods may have timing variations
   - **Mitigation**: Platform-specific calibration, adaptive timing
3. **Permission Issues** - Linux/macOS may require accessibility permissions
   - **Mitigation**: Clear user instructions, graceful permission handling

### Compatibility Concerns
1. **Wayland Support** - Limited automation capabilities
   - **Mitigation**: Fallback to PyAutoGUI, user warnings
2. **macOS Security** - Strict automation restrictions
   - **Mitigation**: Code signing, user guidance for security settings
3. **Display Scaling** - Different handling across platforms
   - **Mitigation**: Platform-specific scaling detection and compensation

## Recommended Implementation Order

### Phase 1: Foundation (1-2 weeks)
```python
1. Create platform abstraction interfaces
2. Implement Windows adapter (refactor existing code)
3. Basic Linux adapter using PyAutoGUI
4. Platform detection and factory pattern
```

### Phase 2: Core Features (2-3 weeks)
```python
1. Linux X11 implementation for better performance
2. Cross-platform configuration system
3. Platform-specific optimizations
4. Comprehensive testing suite
```

### Phase 3: Enhancement (1-2 weeks)
```python
1. Wayland support investigation
2. macOS implementation
3. Performance profiling and optimization
4. Documentation and user guides
```

## Conclusion

While DigTool is currently Windows-exclusive, a well-planned cross-platform adaptation is definitely achievable. The key is to:

1. **Abstract platform-specific operations** behind clean interfaces
2. **Implement progressive platform support** starting with PyAutoGUI
3. **Maintain performance** through platform-specific optimizations
4. **Test thoroughly** across different environments

The estimated effort for basic cross-platform support is **4-6 weeks** for a single developer, with ongoing maintenance for platform-specific issues and optimizations.