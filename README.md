# White Luminance Greyscale Color Inverter

A powerful desktop application for processing images with color inversion, grayscale conversion, and white luminance enhancement effects.

## Features

- **Color Inversion** - Inverts all colors in the image
- **Grayscale Conversion** - Converts to grayscale with optimal luminance mapping
- **White Luminance Aura** - Adds customizable glow effect around bright areas
- **Real-time Preview** - Instant visual feedback with adjustable parameters
- **Batch Processing** - Process entire folders automatically
- **CLI Support** - Use from terminal without GUI
- **Cross-platform** - Works on Windows, macOS, and Linux

## Screenshots

![Main Interface](screenshot.png)

## Installation

### Requirements

- Python 3.8 or higher
- pip package manager

### Setup

```


# Clone or download this repository

cd white-luminance-inverter

# Create virtual environment

python -m venv venv

# Activate virtual environment

# Windows:

venv\Scripts\activate

# macOS/Linux:

source venv/bin/activate

# Install dependencies

pip install -r requirements.txt

```

### Dependencies

```

PyQt6>=6.6.0
Pillow>=10.0.0
numpy>=1.24.0
scipy>=1.10.0

```

## Usage

### GUI Mode (Default)

```

python app.py

```

#### How to Use the GUI

1. **Drag & Drop** images onto the interface or click to browse
2. **Adjust Parameters:**
   - **Aura Slider** (0-50): Controls glow intensity around bright areas
   - **Threshold Slider** (100-250): Sets brightness level for luminance detection
3. **Preview** in real-time - click images to view full size
4. **Export** individual images or batch export all
5. **Auto-Process** - Click "⚡ Auto-Process ./input Folder" to batch process

#### UI Scale

Change interface size from the dropdown menu (Small/Medium/Large/Extra Large) - updates instantly.

### CLI Mode (No GUI)

```


# Process input folder with defaults

python app.py --no-gui

# Custom settings

python app.py --no-gui --aura 25 --threshold 180

# Custom folders

python app.py --no-gui --input ./photos --output ./processed

# All options

python app.py --no-gui --aura 30 --threshold 150 --input ./my_images --output ./results

```

#### CLI Arguments

| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `--no-gui` | flag | False | Run in terminal mode without GUI |
| `--aura` | float | 15 | Aura size (0-50) |
| `--threshold` | int | 200 | White threshold (100-250) |
| `--input` | str | "input" | Input folder path |
| `--output` | str | "output" | Output folder path |

#### CLI Examples

```


# Help

python app.py --help

# Basic batch processing

python app.py --no-gui

# High intensity aura

python app.py --no-gui --aura 40

# Lower threshold for more luminance points

python app.py --no-gui --threshold 150

# Process custom directories

python app.py --no-gui --input ~/Pictures --output ~/Processed

```

## Folder Structure

```

white-luminance-inverter/
├── app.py              \# Main application
├── requirements.txt    \# Python dependencies
├── README.md          \# This file
├── input/             \# Place images here for auto-processing
└── output/            \# Processed images saved here (*_processed.jpg)

```

## Output

All processed images are saved as:
- **Format:** JPEG (95% quality, optimized)
- **Naming:** `originalname_processed.jpg`
- **Location:** `./output` folder (customizable via CLI)

## Building Executables

### Windows (.exe)

```

pip install pyinstaller
pyinstaller --onefile --windowed --name "White-Luminance-Inverter" app.py

```

Find executable in: `dist/White-Luminance-Inverter.exe`

### macOS (.app)

```

pip install pyinstaller
pyinstaller --onefile --windowed --name "White-Luminance-Inverter" app.py

```

Find app bundle in: `dist/White-Luminance-Inverter.app`

### Linux

```

pip install pyinstaller
pyinstaller --onefile --name "white-luminance-inverter" app.py

```

Find binary in: `dist/white-luminance-inverter`

## Keyboard Shortcuts

| Action | Shortcut |
|--------|----------|
| Open Files | Click dropzone or drag & drop |
| View Full Image | Click on any preview |
| Close Full View | Click "Close" or ESC |

## Technical Details

### Processing Pipeline

1. **Color Inversion** - RGB values inverted (255 - value)
2. **Grayscale Conversion** - Weighted RGB to luminance (0.299R + 0.587G + 0.114B)
3. **Luminance Detection** - Identifies bright pixels above threshold
4. **Aura Generation** - Gaussian blur applied to luminance mask
5. **Blending** - Maximum blend of aura and grayscale

### Performance

- Images larger than 2000px are automatically resized for faster processing
- Preview images are cached at 400px for smooth UI
- Batch processing uses optimized numpy operations
- Multi-threading support for responsive GUI

## Troubleshooting

### Installation Issues

**Problem:** Pillow build errors on Windows
```


# Solution: Install without version pinning

pip install --upgrade pip
pip install PyQt6 Pillow numpy scipy

```

**Problem:** Qt platform plugin error
```


# Solution: Reinstall PyQt6

pip uninstall PyQt6 PyQt6-Qt6 PyQt6-sip
pip install PyQt6

```

### Runtime Issues

**Problem:** Images not processing
- Check file format is supported (.png, .jpg, .jpeg, .bmp, .gif, .webp)
- Verify input/output folders exist and have write permissions

**Problem:** Slow processing
- Reduce image resolution before processing
- Lower aura value for faster computation
- Use CLI mode for batch processing (faster than GUI)

## Supported Formats

**Input:** PNG, JPEG, JPG, BMP, GIF, WEBP  
**Output:** JPEG (optimized, 95% quality)

## License

MIT License - Feel free to use and modify

## Credits

Original concept by oa,7  
Python implementation with PyQt6

## Contributing

Contributions welcome! Please open an issue or submit a pull request.

## Changelog

### v1.0.0 (2025-12-17)
- Initial release
- GUI and CLI modes
- Batch processing
- Real-time preview
- Adjustable parameters
- Cross-platform support

## Support

For issues or questions, please open an issue on GitHub.

---

**Made with Python + PyQt6**
