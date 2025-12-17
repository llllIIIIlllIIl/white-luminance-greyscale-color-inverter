import sys
import os
from pathlib import Path
from datetime import datetime
import numpy as np
from PIL import Image
import argparse
from scipy.ndimage import gaussian_filter


# Create folders
os.makedirs("input", exist_ok=True)
os.makedirs("output", exist_ok=True)

# UI Scale settings
UI_SCALES = {
    "Small": 0.75,
    "Medium": 1.0,
    "Large": 1.25,
    "Extra Large": 1.5
}


def pil_to_qpixmap(pil_img):
    """Safe PIL to QPixmap conversion"""
    from PyQt6.QtGui import QPixmap, QImage
    
    if pil_img.mode != "RGB":
        pil_img = pil_img.convert("RGB")
    
    data = pil_img.tobytes("raw", "RGB")
    qimg = QImage(data, pil_img.width, pil_img.height, pil_img.width * 3, QImage.Format.Format_RGB888)
    return QPixmap.fromImage(qimg.copy())


def process_image_with_aura(image_path, aura_size, white_threshold):
    """Fast processing with invert + grayscale + glow"""
    img = Image.open(image_path)
    
    # Resize for speed if too large
    max_size = 2000
    if max(img.size) > max_size:
        img.thumbnail((max_size, max_size), Image.Resampling.LANCZOS)
    
    if img.mode != "RGB":
        img = img.convert("RGB")
    
    # Invert
    arr = np.array(img)
    inverted = 255 - arr
    
    # Grayscale
    gray_arr = np.dot(inverted[...,:3], [0.299, 0.587, 0.114]).astype("uint8")
    
    white_pixels = np.where(gray_arr > white_threshold)
    white_count = len(white_pixels[0])
    
    # Aura effect
    if aura_size > 0 and white_count > 0:
        mask = (gray_arr > white_threshold).astype(float)
        scale = np.sqrt((gray_arr.shape[0] * gray_arr.shape[1]) / (220 * 220))
        sigma = max(0.5, aura_size * scale * 0.2)
        glow = gaussian_filter(mask, sigma=sigma) * 255
        enhanced = np.maximum(gray_arr, glow.astype("uint8"))
        output_arr = np.stack([enhanced, enhanced, enhanced], axis=2)
    else:
        output_arr = np.stack([gray_arr, gray_arr, gray_arr], axis=2)
    
    output = Image.fromarray(output_arr.astype("uint8"))
    return output, white_count


def batch_process_input_folder(aura_size=15, white_threshold=200, input_dir="input", output_dir="output"):
    """Auto-process everything in input folder and save to output folder"""
    input_folder = Path(input_dir)
    output_folder = Path(output_dir)
    
    # Create output folder if needed
    output_folder.mkdir(exist_ok=True)
    
    supported = {".png", ".jpg", ".jpeg", ".bmp", ".gif", ".webp"}
    files = [f for f in input_folder.iterdir() if f.suffix.lower() in supported]
    
    if not files:
        print(f"No images found in ./{input_dir} folder")
        return 0
    
    print(f"\n🔄 Processing {len(files)} images from ./{input_dir}...")
    print(f"   Settings: Aura={aura_size}, Threshold={white_threshold}")
    
    for i, file in enumerate(files, 1):
        try:
            processed, count = process_image_with_aura(str(file), aura_size, white_threshold)
            output_path = output_folder / f"{file.stem}_processed.jpg"
            processed.save(output_path, "JPEG", quality=95, optimize=True)
            print(f"  [{i}/{len(files)}] ✓ {file.name} → {output_path.name} ({count} luminance points)")
        except Exception as e:
            print(f"  [{i}/{len(files)}] ✗ {file.name} - Error: {e}")
    
    print(f"\n✅ Batch processing complete! Check ./{output_dir} folder\n")
    return len(files)


# GUI CODE BELOW - Only imported if not in CLI mode

def run_gui():
    """Run the GUI application"""
    from PyQt6.QtWidgets import (
        QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
        QLabel, QSlider, QPushButton, QFileDialog, QScrollArea, QGridLayout,
        QFrame, QMessageBox, QDialog, QComboBox
    )
    from PyQt6.QtCore import Qt, QSize, QTimer
    from PyQt6.QtGui import QPixmap, QImage, QFont, QDragEnterEvent, QDropEvent, QCursor
    
    class ImageViewerDialog(QDialog):
        """Full-size image viewer"""
        def __init__(self, pixmap, title, parent=None):
            super().__init__(parent)
            self.setWindowTitle(title)
            self.setModal(True)
            
            screen = QApplication.primaryScreen().geometry()
            self.setGeometry(100, 100, int(screen.width() * 0.8), int(screen.height() * 0.8))
            
            layout = QVBoxLayout(self)
            
            scroll = QScrollArea()
            scroll.setWidgetResizable(True)
            scroll.setStyleSheet("QScrollArea { background: #000; border: none; }")
            
            img_label = QLabel()
            img_label.setPixmap(pixmap)
            img_label.setAlignment(Qt.AlignmentFlag.AlignCenter)
            img_label.setStyleSheet("background: #000;")
            
            scroll.setWidget(img_label)
            layout.addWidget(scroll)
            
            close_btn = QPushButton("Close")
            close_btn.setStyleSheet("""
                QPushButton {
                    background:#444;
                    color:#fff;
                    border-radius:8px;
                    padding:10px;
                    font-weight:bold;
                }
                QPushButton:hover { background:#555; }
            """)
            close_btn.clicked.connect(self.close)
            layout.addWidget(close_btn)
    
    class ClickableLabel(QLabel):
        """Label that opens full image on click"""
        def __init__(self, parent=None):
            super().__init__(parent)
            self.full_pixmap = None
            self.image_title = "Image"
            self.setCursor(QCursor(Qt.CursorShape.PointingHandCursor))
            self.setStyleSheet("QLabel { border: none; }")
        
        def set_full_image(self, pixmap, title):
            self.full_pixmap = pixmap
            self.image_title = title
        
        def mousePressEvent(self, event):
            if self.full_pixmap and event.button() == Qt.MouseButton.LeftButton:
                dialog = ImageViewerDialog(self.full_pixmap, self.image_title, self)
                dialog.exec()
    
    class ImageCard(QFrame):
        def __init__(self, file_path, scale=1.0, parent=None):
            super().__init__(parent)
            self.file_path = file_path
            self.scale = scale
            self.debounce_timer = QTimer(self)
            self.debounce_timer.setSingleShot(True)
            self.debounce_timer.timeout.connect(self._process)
            
            self.setStyleSheet("""
                QFrame {
                    background: rgba(255,255,255,0.03);
                    border-radius: 10px;
                    border: 1px solid rgba(255,255,255,0.08);
                    padding: 15px;
                }
            """)
            
            self._build_ui()
            self._load_and_process()
        
        def _build_ui(self):
            layout = QVBoxLayout(self)
            
            self.header = QLabel(f"⚫ {Path(self.file_path).name}")
            self.header.setFont(QFont("Segoe UI", int(10 * self.scale), QFont.Weight.Bold))
            self.header.setStyleSheet("color: #bbb;")
            layout.addWidget(self.header)
            
            imgs = QHBoxLayout()
            
            orig_frame = QFrame()
            orig_frame.setStyleSheet("background:#1a1a1a;border-radius:8px;")
            orig_v = QVBoxLayout(orig_frame)
            self.orig_label = QLabel("Original (click to view)")
            self.orig_label.setStyleSheet("color:#888;font-size:10px;")
            orig_v.addWidget(self.orig_label)
            self.orig_img = ClickableLabel()
            self.orig_img.setMinimumSize(int(200 * self.scale), int(200 * self.scale))
            self.orig_img.setAlignment(Qt.AlignmentFlag.AlignCenter)
            orig_v.addWidget(self.orig_img)
            imgs.addWidget(orig_frame)
            
            proc_frame = QFrame()
            proc_frame.setStyleSheet("background:#1a1a1a;border-radius:8px;")
            proc_v = QVBoxLayout(proc_frame)
            self.status_lbl = QLabel("Processing...")
            self.status_lbl.setStyleSheet("color:#888;")
            proc_v.addWidget(self.status_lbl)
            self.proc_img = ClickableLabel()
            self.proc_img.setMinimumSize(int(200 * self.scale), int(200 * self.scale))
            self.proc_img.setAlignment(Qt.AlignmentFlag.AlignCenter)
            proc_v.addWidget(self.proc_img)
            imgs.addWidget(proc_frame)
            
            layout.addLayout(imgs)
            
            ctrl = QHBoxLayout()
            
            ctrl.addWidget(QLabel("Aura:"))
            self.aura_slider = QSlider(Qt.Orientation.Horizontal)
            self.aura_slider.setRange(0, 50)
            self.aura_slider.setValue(15)
            self.aura_val = QLabel("15")
            self.aura_slider.valueChanged.connect(self._slider_changed)
            ctrl.addWidget(self.aura_slider)
            ctrl.addWidget(self.aura_val)
            
            ctrl.addWidget(QLabel("Threshold:"))
            self.thr_slider = QSlider(Qt.Orientation.Horizontal)
            self.thr_slider.setRange(100, 250)
            self.thr_slider.setValue(200)
            self.thr_val = QLabel("200")
            self.thr_slider.valueChanged.connect(self._slider_changed)
            ctrl.addWidget(self.thr_slider)
            ctrl.addWidget(self.thr_val)
            
            layout.addLayout(ctrl)
            
            self.dl_btn = QPushButton("⬇️ Export to ./output")
            self.dl_btn.setStyleSheet("""
                QPushButton {
                    background:#444;
                    color:#fff;
                    border-radius:8px;
                    padding:8px;
                    font-weight:bold;
                }
                QPushButton:hover { background:#555; }
            """)
            self.dl_btn.clicked.connect(self._download)
            layout.addWidget(self.dl_btn)
        
        def update_scale(self, new_scale):
            self.scale = new_scale
            self.header.setFont(QFont("Segoe UI", int(10 * self.scale), QFont.Weight.Bold))
            self.orig_img.setMinimumSize(int(200 * self.scale), int(200 * self.scale))
            self.proc_img.setMinimumSize(int(200 * self.scale), int(200 * self.scale))
            if self.orig_img.pixmap():
                self._rescale_preview()
        
        def _rescale_preview(self):
            try:
                img = Image.open(self.file_path)
                img.thumbnail((400, 400), Image.Resampling.LANCZOS)
                pixmap = pil_to_qpixmap(img)
                scaled = pixmap.scaled(int(200 * self.scale), int(200 * self.scale), 
                                      Qt.AspectRatioMode.KeepAspectRatio, 
                                      Qt.TransformationMode.FastTransformation)
                self.orig_img.setPixmap(scaled)
                
                if self.proc_img.full_pixmap:
                    scaled_proc = self.proc_img.full_pixmap.scaled(
                        int(200 * self.scale), int(200 * self.scale),
                        Qt.AspectRatioMode.KeepAspectRatio,
                        Qt.TransformationMode.FastTransformation
                    )
                    self.proc_img.setPixmap(scaled_proc)
            except:
                pass
        
        def _load_and_process(self):
            try:
                img = Image.open(self.file_path)
                full_pixmap = pil_to_qpixmap(img)
                
                img.thumbnail((400, 400), Image.Resampling.LANCZOS)
                pixmap = pil_to_qpixmap(img)
                scaled = pixmap.scaled(int(200 * self.scale), int(200 * self.scale), 
                                      Qt.AspectRatioMode.KeepAspectRatio, 
                                      Qt.TransformationMode.FastTransformation)
                self.orig_img.setPixmap(scaled)
                self.orig_img.set_full_image(full_pixmap, f"Original - {Path(self.file_path).name}")
                
                self._process()
            except Exception as e:
                self.status_lbl.setText(f"Error: {e}")
        
        def _slider_changed(self):
            self.aura_val.setText(str(self.aura_slider.value()))
            self.thr_val.setText(str(self.thr_slider.value()))
            self.status_lbl.setText("Processing...")
            self.debounce_timer.start(500)
        
        def _process(self):
            try:
                processed, count = process_image_with_aura(
                    self.file_path,
                    self.aura_slider.value(),
                    self.thr_slider.value()
                )
                
                full_pixmap = pil_to_qpixmap(processed)
                
                pixmap = full_pixmap
                scaled = pixmap.scaled(int(200 * self.scale), int(200 * self.scale),
                                      Qt.AspectRatioMode.KeepAspectRatio,
                                      Qt.TransformationMode.FastTransformation)
                self.proc_img.setPixmap(scaled)
                self.proc_img.set_full_image(full_pixmap, f"Processed - {Path(self.file_path).name}")
                self.status_lbl.setText(f"⚫ {count} luminance points")
            except Exception as e:
                self.status_lbl.setText(f"Error: {e}")
        
        def _download(self):
            try:
                # Load full resolution
                processed, _ = process_image_with_aura(
                    self.file_path,
                    self.aura_slider.value(),
                    self.thr_slider.value()
                )
                
                base = Path(self.file_path).stem
                output_path = Path("output") / f"{base}_processed.jpg"
                processed.save(output_path, "JPEG", quality=95, optimize=True)
                
                self.dl_btn.setText("✅ Exported!")
                QTimer.singleShot(2000, lambda: self.dl_btn.setText("⬇️ Export to ./output"))
                
            except Exception as e:
                QMessageBox.critical(self, "Error", str(e))
    
    class DropZone(QFrame):
        def __init__(self, scale=1.0, parent=None):
            super().__init__(parent)
            self.parent_app = parent
            self.scale = scale
            self.setAcceptDrops(True)
            self.setMinimumHeight(int(120 * self.scale))
            self.setAttribute(Qt.WidgetAttribute.WA_StyledBackground, True)
            self.setStyleSheet("""
                DropZone {
                    background:#333;
                    border:3px dashed #666;
                    border-radius:15px;
                }
                DropZone:hover {
                    background:#444;
                    border-color:#888;
                }
            """)
            
            self.layout = QVBoxLayout(self)
            self.layout.setAlignment(Qt.AlignmentFlag.AlignCenter)
            
            self.icon = QLabel("⬆️")
            self.icon.setFont(QFont("Segoe UI", int(48 * self.scale)))
            self.icon.setAlignment(Qt.AlignmentFlag.AlignCenter)
            self.icon.setStyleSheet("color:#ccc;background:transparent;border:none;")
            self.icon.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
            self.layout.addWidget(self.icon)
            
            self.text = QLabel("Drag & Drop Images Here")
            self.text.setFont(QFont("Segoe UI", int(14 * self.scale), QFont.Weight.Bold))
            self.text.setAlignment(Qt.AlignmentFlag.AlignCenter)
            self.text.setStyleSheet("color:#ccc;background:transparent;border:none;")
            self.text.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
            self.layout.addWidget(self.text)
            
            self.subtext = QLabel("or click to browse")
            self.subtext.setFont(QFont("Segoe UI", int(11 * self.scale)))
            self.subtext.setAlignment(Qt.AlignmentFlag.AlignCenter)
            self.subtext.setStyleSheet("color:#888;background:transparent;border:none;")
            self.subtext.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
            self.layout.addWidget(self.subtext)
        
        def update_scale(self, new_scale):
            self.scale = new_scale
            self.setMinimumHeight(int(120 * self.scale))
            self.icon.setFont(QFont("Segoe UI", int(48 * self.scale)))
            self.text.setFont(QFont("Segoe UI", int(14 * self.scale), QFont.Weight.Bold))
            self.subtext.setFont(QFont("Segoe UI", int(11 * self.scale)))
        
        def dragEnterEvent(self, event):
            if event.mimeData().hasUrls():
                event.acceptProposedAction()
                self.setStyleSheet("""
                    DropZone {
                        background:#555;
                        border:3px dashed #aaa;
                        border-radius:15px;
                    }
                """)
        
        def dragLeaveEvent(self, event):
            self.setStyleSheet("""
                DropZone {
                    background:#333;
                    border:3px dashed #666;
                    border-radius:15px;
                }
                DropZone:hover {
                    background:#444;
                    border-color:#888;
                }
            """)
        
        def dropEvent(self, event):
            files = []
            for url in event.mimeData().urls():
                file_path = url.toLocalFile()
                if Path(file_path).suffix.lower() in {'.png', '.jpg', '.jpeg', '.bmp', '.gif', '.webp'}:
                    files.append(file_path)
            
            if files and self.parent_app:
                self.parent_app.add_files(files)
            
            self.setStyleSheet("""
                DropZone {
                    background:#333;
                    border:3px dashed #666;
                    border-radius:15px;
                }
                DropZone:hover {
                    background:#444;
                    border-color:#888;
                }
            """)
            event.acceptProposedAction()
        
        def mousePressEvent(self, event):
            if self.parent_app:
                self.parent_app._select()
    
    class VisualIdentityApp(QMainWindow):
        def __init__(self):
            super().__init__()
            self.current_scale = 1.0
            self.setWindowTitle("White Luminance Greyscale Color Inverter")
            self.setGeometry(100, 100, int(1200 * self.current_scale), int(800 * self.current_scale))
            self.cards = []
            self.setAcceptDrops(True)
            
            self.setStyleSheet("""
                QMainWindow { 
                    background: #1a1a1a; 
                } 
                QLabel { 
                    color: #e0e0e0; 
                }
                * {
                    outline: none;
                }
            """)
            
            central = QWidget()
            self.setCentralWidget(central)
            self.main_layout = QVBoxLayout(central)
            
            self._build_ui()
        
        def _build_ui(self):
            header = QHBoxLayout()
            
            self.title = QLabel("White Luminance Greyscale Color Inverter")
            self.title.setFont(QFont("Segoe UI", int(22 * self.current_scale), QFont.Weight.Bold))
            self.title.setStyleSheet("color:#fff;")
            header.addWidget(self.title)
            
            header.addStretch()
            
            scale_label = QLabel("UI Scale:")
            scale_label.setStyleSheet("color:#aaa;")
            header.addWidget(scale_label)
            
            self.scale_combo = QComboBox()
            self.scale_combo.addItems(UI_SCALES.keys())
            self.scale_combo.setCurrentText("Medium")
            self.scale_combo.setStyleSheet("""
                QComboBox {
                    background:#333;
                    color:#fff;
                    border-radius:5px;
                    padding:5px;
                    min-width:100px;
                }
                QComboBox:hover { background:#444; }
            """)
            self.scale_combo.currentTextChanged.connect(self._change_scale)
            header.addWidget(self.scale_combo)
            
            self.main_layout.addLayout(header)
            
            self.subtitle = QLabel("color inversion + grayscale + white luminance")
            self.subtitle.setAlignment(Qt.AlignmentFlag.AlignCenter)
            self.subtitle.setStyleSheet("color:#888;font-size:12px;")
            self.main_layout.addWidget(self.subtitle)
            
            self.batch_btn = QPushButton("⚡ Auto-Process ./input Folder")
            self.batch_btn.setMinimumHeight(int(50 * self.current_scale))
            self.batch_btn.setFont(QFont("Segoe UI", int(12 * self.current_scale), QFont.Weight.Bold))
            self.batch_btn.setStyleSheet("""
                QPushButton {
                    background:#2a5;
                    border-radius:10px;
                    color:#fff;
                }
                QPushButton:hover { background:#3b6; }
            """)
            self.batch_btn.clicked.connect(self._batch_process)
            self.main_layout.addWidget(self.batch_btn)
            
            self.dropzone = DropZone(self.current_scale, self)
            self.main_layout.addWidget(self.dropzone)
            
            scroll = QScrollArea()
            scroll.setWidgetResizable(True)
            scroll.setStyleSheet("QScrollArea { border:none; background:transparent; }")
            self.cards_widget = QWidget()
            self.cards_layout = QGridLayout(self.cards_widget)
            scroll.setWidget(self.cards_widget)
            self.main_layout.addWidget(scroll)
            
            self.export_btn = QPushButton("⬛ Export All to ./output")
            self.export_btn.setMinimumHeight(int(50 * self.current_scale))
            self.export_btn.setStyleSheet("QPushButton { background:#333; color:#fff; border-radius:10px; }")
            self.export_btn.clicked.connect(self._export)
            self.export_btn.setVisible(False)
            self.main_layout.addWidget(self.export_btn)
        
        def _change_scale(self, scale_name):
            new_scale = UI_SCALES[scale_name]
            if new_scale == self.current_scale:
                return
            
            self.current_scale = new_scale
            self.title.setFont(QFont("Segoe UI", int(22 * self.current_scale), QFont.Weight.Bold))
            self.batch_btn.setMinimumHeight(int(50 * self.current_scale))
            self.batch_btn.setFont(QFont("Segoe UI", int(12 * self.current_scale), QFont.Weight.Bold))
            self.export_btn.setMinimumHeight(int(50 * self.current_scale))
            self.dropzone.update_scale(new_scale)
            
            for card in self.cards:
                card.update_scale(new_scale)
            
            self.resize(int(1200 * self.current_scale), int(800 * self.current_scale))
        
        def dragEnterEvent(self, event):
            if event.mimeData().hasUrls():
                event.acceptProposedAction()
        
        def dropEvent(self, event):
            files = []
            for url in event.mimeData().urls():
                file_path = url.toLocalFile()
                if Path(file_path).suffix.lower() in {'.png', '.jpg', '.jpeg', '.bmp', '.gif', '.webp'}:
                    files.append(file_path)
            
            if files:
                self.add_files(files)
            
            event.acceptProposedAction()
        
        def add_files(self, file_paths):
            for path in file_paths:
                card = ImageCard(path, self.current_scale)
                row = len(self.cards) // 2
                col = len(self.cards) % 2
                self.cards_layout.addWidget(card, row, col)
                self.cards.append(card)
            
            if self.cards:
                self.export_btn.setVisible(True)
        
        def _batch_process(self):
            count = batch_process_input_folder()
            if count > 0:
                QMessageBox.information(self, "Done", f"Processed {count} images!\nCheck ./output folder")
            else:
                QMessageBox.information(self, "Empty", "No images in ./input folder")
        
        def _select(self):
            paths, _ = QFileDialog.getOpenFileNames(
                self, "Select Images", "", "Images (*.png *.jpg *.jpeg *.bmp)"
            )
            
            if paths:
                self.add_files(paths)
        
        def _export(self):
            try:
                for i, card in enumerate(self.cards, 1):
                    processed, _ = process_image_with_aura(
                        card.file_path,
                        card.aura_slider.value(),
                        card.thr_slider.value()
                    )
                    base = Path(card.file_path).stem
                    output_path = Path("output") / f"{base}_processed.jpg"
                    processed.save(output_path, "JPEG", quality=95, optimize=True)
                
                QMessageBox.information(self, "Done", f"Exported {len(self.cards)} images to ./output")
            except Exception as e:
                QMessageBox.critical(self, "Error", str(e))
    
    app = QApplication(sys.argv)
    win = VisualIdentityApp()
    win.show()
    sys.exit(app.exec())


if __name__ == "__main__":
    parser = argparse.ArgumentParser(
        description="White Luminance Greyscale Color Inverter - Color inversion + grayscale + white luminance",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  # Open GUI (default)
  python app.py
  
  # Process input folder without GUI
  python app.py --no-gui
  
  # Custom settings
  python app.py --no-gui --aura 25 --threshold 180
  
  # Custom folders
  python app.py --no-gui --input ./photos --output ./processed
        """
    )
    
    parser.add_argument("--no-gui", action="store_true", 
                       help="Run in terminal mode without GUI")
    parser.add_argument("--aura", type=float, default=15,
                       help="Aura size (0-50, default: 15)")
    parser.add_argument("--threshold", type=int, default=200,
                       help="White threshold (100-250, default: 200)")
    parser.add_argument("--input", type=str, default="input",
                       help="Input folder path (default: ./input)")
    parser.add_argument("--output", type=str, default="output",
                       help="Output folder path (default: ./output)")
    
    args = parser.parse_args()
    
    if args.no_gui:
        # CLI mode
        batch_process_input_folder(
            aura_size=args.aura,
            white_threshold=args.threshold,
            input_dir=args.input,
            output_dir=args.output
        )
    else:
        # GUI mode - auto-process on startup then open GUI
        batch_process_input_folder()
        run_gui()
