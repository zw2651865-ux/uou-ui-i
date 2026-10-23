# -*- coding: utf-8 -*-
"""PromptDesk - 本地提示词管理器（PySide6）
启动：py -3 -m pip install PySide6
     py -3 promptdesk.pyw
"""
import sys, os, json, shutil, traceback, uuid
from datetime import datetime
from pathlib import Path
from PySide6.QtCore import Qt, QSize, QPoint, QRect
from PySide6.QtGui import QAction, QKeySequence, QFont, QPainter, QColor, QPen, QBrush, QIcon, QPixmap, QTextLayout, QTextOption
from PySide6.QtWidgets import QStyledItemDelegate, QStyle
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout, QSplitter,
    QListWidget, QListWidgetItem, QLineEdit, QPushButton, QLabel, QMenu,
    QMessageBox, QDialog, QFormLayout, QDialogButtonBox, QTextEdit, QComboBox,
    QInputDialog, QFileDialog, QStatusBar, QToolButton, QColorDialog, QGridLayout, QSpinBox, QFontComboBox
)

APP_NAME = "PromptDesk"
DATA_DIR = Path(os.getenv("APPDATA", str(Path.home()))) / "PromptDesk"
DATA_FILE = DATA_DIR / "prompts.json"
LOG_FILE = DATA_DIR / "error.log"
BACKUP_DIR = DATA_DIR / "backups"
HIDDEN_CATEGORIES_FILE = DATA_DIR / "hidden_categories.json"
SETTINGS_FILE = DATA_DIR / "settings.json"
DEFAULT_CATEGORIES = ["全部提示词", "写作", "图片生成", "视频生成", "翻译", "工作"]


def log_error():
    try:
        DATA_DIR.mkdir(parents=True, exist_ok=True)
        with LOG_FILE.open("a", encoding="utf-8") as f:
            f.write("\n" + "=" * 60 + "\n" + datetime.now().isoformat() + "\n")
            traceback.print_exc(file=f)
    except Exception:
        pass


class PromptListWidget(QListWidget):
    """支持鼠标框选多张卡片，并识别拖动，避免拖选时误复制。"""
    def __init__(self, parent=None):
        super().__init__(parent)
        self.drag_selecting = False
        self._press_pos = QPoint()

    def mousePressEvent(self, event):
        self._press_pos = event.position().toPoint()
        self.drag_selecting = False
        super().mousePressEvent(event)

    def mouseMoveEvent(self, event):
        if event.buttons() & Qt.MouseButton.LeftButton:
            if (event.position().toPoint() - self._press_pos).manhattanLength() >= QApplication.startDragDistance():
                self.drag_selecting = True
        super().mouseMoveEvent(event)

    def mouseReleaseEvent(self, event):
        super().mouseReleaseEvent(event)
        # 等 itemClicked 发出后再清除拖动状态，避免框选操作触发复制。
        if self.drag_selecting:
            from PySide6.QtCore import QTimer
            QTimer.singleShot(0, self._clear_drag_state)

    def _clear_drag_state(self):
        self.drag_selecting = False


def new_prompt(title="新提示词", content="", category="未分类"):
    return {"id": str(uuid.uuid4()), "title": title, "content": content,
            "category": category, "favorite": False, "pinned": False,
            "deleted": False, "created": datetime.now().isoformat(timespec="seconds"),
            "updated": datetime.now().isoformat(timespec="seconds"), "used": ""}


class PromptCardDelegate(QStyledItemDelegate):
    """绘制规则的横向圆角矩形提示词卡片。"""
    def paint(self, painter, option, index):
        painter.save()
        rect = option.rect.adjusted(4, 6, -4, -6)
        selected = bool(option.state & QStyle.StateFlag.State_Selected)
        hovered = bool(option.state & QStyle.StateFlag.State_MouseOver)
        theme = getattr(self, "theme", "深色")
        palettes = {
            "深色": (("#293b2a", "#83c64a", "#c8ff9d"), ("#222a32", "#526345", "#f0f5fa"), ("#1b222b", "#303945", "#e7edf3")),
            "浅色": (("#dcecff", "#3478d4", "#163c6a"), ("#f0f6ff", "#8ab5e8", "#202833"), ("#ffffff", "#d1d9e2", "#202833")),
            "柔和绿": (("#d9ead9", "#3c8b58", "#214b2d"), ("#eaf4e8", "#8ab995", "#24352a"), ("#f5faf3", "#c4d6c3", "#24352a")),
        }
        selected_colors, hover_colors, normal_colors = palettes.get(theme, palettes["深色"])
        bg, border, fg = map(QColor, selected_colors if selected else hover_colors if hovered else normal_colors)
        custom_fg = getattr(self, "custom_text_color", "")
        if custom_fg:
            fg = QColor(custom_fg)
        painter.setRenderHint(QPainter.RenderHint.Antialiasing, True)
        painter.setPen(QPen(border, 1))
        painter.setBrush(QBrush(bg))
        painter.drawRoundedRect(rect, 9, 9)
        text_rect = rect.adjusted(7, 7, -7, -7)
        category = index.data(Qt.ItemDataRole.UserRole + 2) or "未分类"
        content = index.data(Qt.ItemDataRole.UserRole + 1) or "（没有提示词内容）"

        # 分类作为小标签显示，卡片主体直接展示提示词内容，不显示标题。
        accent = {"深色":"#a3ff4f", "浅色":"#3478d4", "柔和绿":"#3c8b58"}.get(getattr(self, "theme", "深色"), "#a3ff4f")
        painter.setPen(QColor(accent))
        cat_font = painter.font()
        cat_font.setPointSize(8)
        cat_font.setWeight(QFont.Weight.DemiBold)
        painter.setFont(cat_font)
        painter.drawText(text_rect, int(Qt.AlignmentFlag.AlignTop | Qt.AlignmentFlag.AlignLeft), category)

        content_rect = text_rect.adjusted(0, 19, 0, 0)
        painter.setPen(fg)
        font = painter.font()
        font.setPointSize(max(8, int(getattr(self, "prompt_font_size", 12)) - 3))
        font.setFamily(getattr(self, "prompt_font_family", ""))
        font.setWeight(getattr(self, "prompt_font_weight", QFont.Weight.Normal))
        painter.setFont(font)
        metrics = painter.fontMetrics()
        max_width = max(10, content_rect.width())
        available_height = max(1, content_rect.height())
        line_height = max(1, metrics.lineSpacing())
        max_lines = max(1, available_height // line_height)

        # QTextLayout 负责可靠的自动换行，避免手动逐字符绘制造成内容不显示。
        # 保留原始换行；空间不足时只在卡片视觉上截断，不改动原始提示词。
        display_content = str(content).replace("\r\n", "\n").replace("\r", "\n")
        if not display_content.strip():
            display_content = "（没有提示词内容）"
        text_option = QTextOption()
        text_option.setWrapMode(QTextOption.WrapMode.WrapAtWordBoundaryOrAnywhere)
        text_layout = QTextLayout(display_content, font)
        text_layout.setTextOption(text_option)
        text_layout.beginLayout()
        text_lines = []
        while True:
            text_line = text_layout.createLine()
            if not text_line.isValid():
                break
            text_line.setLineWidth(max_width)
            text_lines.append(text_line)
        text_layout.endLayout()

        visible_lines = text_lines[:max_lines]
        overflow = len(text_lines) > max_lines
        for line_no, text_line in enumerate(visible_lines):
            line_text = display_content[text_line.textStart():text_line.textStart() + text_line.textLength()]
            if overflow and line_no == len(visible_lines) - 1:
                line_text = metrics.elidedText(line_text.rstrip() + "…", Qt.TextElideMode.ElideRight, max_width)
            y = content_rect.top() + line_no * line_height
            if y + metrics.height() > content_rect.bottom() + 1:
                break
            painter.drawText(QRect(content_rect.left(), y, content_rect.width(), metrics.height()),
                             int(Qt.AlignmentFlag.AlignTop | Qt.AlignmentFlag.AlignLeft), line_text)
        painter.restore()

    def sizeHint(self, option, index):
        return QSize(306, 116)


class PromptDialog(QDialog):
    def __init__(self, parent=None, prompt=None, categories=None):
        super().__init__(parent)
        self.setWindowTitle("编辑提示词" if prompt else "新建提示词")
        self.resize(620, 520)
        self.prompt = prompt or {}
        layout = QVBoxLayout(self)
        form = QFormLayout()
        # 提示词编辑界面不再要求填写标题；旧数据中的标题仅作内部兼容。
        self.category_combo = QComboBox()
        category_options = ["未分类"] + [c for c in (categories or []) if c not in ("全部提示词", "未分类")]
        self.category_combo.addItems(list(dict.fromkeys(category_options)))
        category = self.prompt.get("category", "未分类")
        idx = self.category_combo.findText(category)
        if idx >= 0:
            self.category_combo.setCurrentIndex(idx)
        self.content_edit = QTextEdit()
        self.content_edit.setPlaceholderText("在这里输入提示词内容……")
        self.content_edit.setPlainText(self.prompt.get("content", ""))
        self.annotation_edit = QLineEdit()
        self.annotation_edit.setPlaceholderText("可选，例如：常用、待优化、菲律宾祈祷视频")
        self.annotation_edit.setText(self.prompt.get("annotation", ""))
        form.addRow("分类", self.category_combo)
        form.addRow("标注 / 备注", self.annotation_edit)
        layout.addLayout(form)
        layout.addWidget(QLabel("提示词内容"))
        layout.addWidget(self.content_edit, 1)
        buttons = QDialogButtonBox(QDialogButtonBox.StandardButton.Save | QDialogButtonBox.StandardButton.Cancel)
        buttons.accepted.connect(self.accept)
        buttons.rejected.connect(self.reject)
        layout.addWidget(buttons)

    def values(self):
        # 保留已有内部标题以兼容旧数据，但不在编辑窗口中显示或要求填写。
        title = self.prompt.get("title", "新提示词") or "新提示词"
        return title, self.content_edit.toPlainText(), self.category_combo.currentText(), self.annotation_edit.text().strip()

    def accept(self):
        _, content, _, _ = self.values()
        if not content.strip():
            answer = QMessageBox.question(self, "内容为空", "提示词内容为空，仍要保存吗？")
            if answer != QMessageBox.StandardButton.Yes:
                return
        super().accept()



class BackgroundWidget(QWidget):
    """绘制铺满可用窗口、保持原始比例的背景图；超出部分居中裁切，不拉伸变形。"""
    def __init__(self, parent=None):
        super().__init__(parent)
        self.background_pixmap = QPixmap()
        self.background_color = QColor("#101318")

    def set_background(self, image_path="", color="#101318"):
        self.background_color = QColor(color)
        self.background_pixmap = QPixmap(image_path) if image_path and Path(image_path).is_file() else QPixmap()
        self.update()

    def paintEvent(self, event):
        painter = QPainter(self)
        painter.fillRect(self.rect(), self.background_color)
        pix = self.background_pixmap
        if not pix.isNull() and self.width() > 0 and self.height() > 0:
            # 先按窗口尺寸等比放大，再从中心裁切到窗口比例：铺满且不扭曲。
            scaled = pix.scaled(self.size(), Qt.AspectRatioMode.KeepAspectRatioByExpanding,
                                Qt.TransformationMode.SmoothTransformation)
            x = max(0, (scaled.width() - self.width()) // 2)
            y = max(0, (scaled.height() - self.height()) // 2)
            painter.drawPixmap(self.rect(), scaled, scaled.rect().adjusted(x, y, -x, -y))
        painter.end()


class PromptDesk(QMainWindow):
    def __init__(self):
        super().__init__()
        DATA_DIR.mkdir(parents=True, exist_ok=True)
        BACKUP_DIR.mkdir(parents=True, exist_ok=True)
        self.items = []
        self.hidden_categories = set()
        self.load_hidden_categories()
        self.current_category = "全部提示词"
        self.current_prompt_id = None
        self._loading = False
        self.settings = self.load_settings()
        self.load_data()
        self.init_ui()
        self.refresh_categories()
        self.refresh_list()

    def init_ui(self):
        self.setWindowTitle("PromptDesk · 提示词管理器")
        self.resize(1100, 700)
        self.setMinimumSize(780, 480)
        self.setStatusBar(QStatusBar(self))
        self.statusBar().showMessage("就绪 · 单击提示词即可复制")

        central = BackgroundWidget()
        central.setObjectName("centralBackground")
        self.central_widget = central
        self.setCentralWidget(central)
        root = QVBoxLayout(central)
        root.setContentsMargins(14, 14, 14, 14)
        root.setSpacing(10)

        top = QHBoxLayout()
        title = QLabel("PROMPTDESK")
        title.setObjectName("brand")
        subtitle = QLabel("本地提示词库")
        subtitle.setObjectName("muted")
        # 固定在顶部工具栏的分类栏开关：隐藏侧栏后按钮仍然可见。
        cat_more = QToolButton()
        cat_more.setText("◧")
        cat_more.setObjectName("categoryPanelToggle")
        cat_more.setCheckable(True)
        cat_more.setChecked(True)
        cat_more.setToolTip("隐藏左侧分类栏（右键打开分类管理）")
        cat_more.setAccessibleName("分类栏显示/隐藏")
        cat_more.setCursor(Qt.CursorShape.PointingHandCursor)
        cat_more.setFixedSize(36, 34)
        cat_more.setStyleSheet("""
            QToolButton#categoryPanelToggle {
                background: rgba(120, 140, 160, 28);
                color: palette(text);
                border: 1px solid rgba(140, 155, 170, 110);
                border-radius: 9px;
                font-size: 18px;
                font-weight: 600;
                padding: 0;
            }
            QToolButton#categoryPanelToggle:hover {
                background: rgba(100, 160, 190, 65);
                border-color: #62a9c8;
            }
            QToolButton#categoryPanelToggle:checked {
                background: rgba(90, 150, 115, 45);
                border-color: rgba(90, 170, 120, 180);
            }
        """)
        cat_more.clicked.connect(self.toggle_category_panel)
        cat_more.setContextMenuPolicy(Qt.ContextMenuPolicy.CustomContextMenu)
        cat_more.customContextMenuRequested.connect(lambda pos: self.category_menu(cat_more.mapToGlobal(cat_more.rect().bottomLeft())))
        self.category_toggle_btn = cat_more
        top.addWidget(cat_more)
        top.addWidget(title)
        top.addWidget(subtitle)
        top.addStretch(1)
        self.appearance_btn = QToolButton()
        self.appearance_btn.setText("🎨 外观设置")
        self.appearance_btn.clicked.connect(self.appearance_menu)
        top.addWidget(self.appearance_btn)
        self.search_edit = QLineEdit()
        self.search_edit.setPlaceholderText("搜索提示词内容或分类…  Ctrl+F")
        self.search_edit.setClearButtonEnabled(True)
        self.search_edit.setFixedWidth(310)
        self.search_edit.textChanged.connect(self.refresh_list)
        top.addWidget(self.search_edit)
        root.addLayout(top)

        splitter = QSplitter(Qt.Orientation.Horizontal)
        root.addWidget(splitter, 1)
        self.main_splitter = splitter

        left_panel = QWidget()
        self.left_panel = left_panel
        left_layout = QVBoxLayout(left_panel)
        left_layout.setContentsMargins(0, 0, 0, 0)
        cat_head = QHBoxLayout()
        cat_label = QLabel("分类")
        cat_label.setObjectName("sectionTitle")
        cat_head.addWidget(cat_label)
        cat_head.addStretch(1)
        left_layout.addLayout(cat_head)
        self.category_list = QListWidget()
        self.category_list.setContextMenuPolicy(Qt.ContextMenuPolicy.CustomContextMenu)
        self.category_list.customContextMenuRequested.connect(self.category_context_menu)
        self.category_list.currentItemChanged.connect(self.on_category_changed)
        left_layout.addWidget(self.category_list, 1)
        splitter.addWidget(left_panel)

        right_panel = QWidget()
        right_layout = QVBoxLayout(right_panel)
        # 仅增加右侧内容区与左侧分类栏之间的距离，不改变提示词卡片尺寸或卡片间距。
        right_layout.setContentsMargins(16, 0, 0, 0)
        list_head = QHBoxLayout()
        list_head.addWidget(QLabel("提示词列表"))
        self.count_label = QLabel("0 条")
        self.count_label.setObjectName("muted")
        list_head.addWidget(self.count_label)
        list_head.addStretch(1)
        self.add_btn = QPushButton("＋ 新建")
        self.add_btn.clicked.connect(self.create_prompt)
        list_head.addWidget(self.add_btn)
        self.batch_delete_btn = QPushButton("批量删除")
        self.batch_delete_btn.setToolTip("先按住鼠标左键拖动框选多张提示词卡片，再点击此按钮")
        self.batch_delete_btn.clicked.connect(self.delete_selected_prompts)
        list_head.addWidget(self.batch_delete_btn)
        right_layout.addLayout(list_head)
        self.prompt_list = PromptListWidget()
        self.prompt_list.setSelectionMode(QListWidget.SelectionMode.ExtendedSelection)
        self.prompt_list.setContextMenuPolicy(Qt.ContextMenuPolicy.CustomContextMenu)
        self.prompt_list.customContextMenuRequested.connect(self.prompt_context_menu)
        # 用明确的鼠标单击事件复制；刷新/自动选中列表项不会触发复制。
        self.prompt_list.itemClicked.connect(self.on_prompt_clicked)
        self.prompt_list.itemDoubleClicked.connect(self.edit_prompt_item)
        self.prompt_list.setSpacing(int(self.settings.get("card_spacing", 10)))
        self.prompt_list.setViewMode(QListWidget.ViewMode.IconMode)
        self.prompt_list.setFlow(QListWidget.Flow.LeftToRight)
        self.prompt_list.setWrapping(True)
        self.prompt_list.setResizeMode(QListWidget.ResizeMode.Adjust)
        self.prompt_list.setMovement(QListWidget.Movement.Static)
        self.prompt_list.setGridSize(QSize(
            int(self.settings.get("card_width", 326)),
            int(self.settings.get("card_height", 132))
        ))
        self.prompt_list.setWordWrap(False)
        self.prompt_list.setUniformItemSizes(True)
        self.card_delegate = PromptCardDelegate(self.prompt_list)
        self.card_delegate.custom_text_color = self.settings.get("card_text_color", "")
        self.card_delegate.prompt_font_size = int(self.settings.get("font_size", 13))
        self.card_delegate.prompt_font_weight = QFont.Weight(int(self.settings.get("font_weight", 400)))
        self.card_delegate.prompt_font_family = self.settings.get("font_family", "")
        self.prompt_list.setItemDelegate(self.card_delegate)
        self.prompt_list.setMouseTracking(True)
        right_layout.addWidget(self.prompt_list, 1)
        tip = QLabel("单击自动复制 · 双击编辑 · 右键查看更多操作")
        tip.setObjectName("muted")
        right_layout.addWidget(tip)
        splitter.addWidget(right_panel)
        splitter.setSizes([240, 820])

        self.apply_theme()

        # Keyboard shortcuts
        self.shortcut("Ctrl+N", self.create_prompt)
        self.shortcut("Ctrl+F", lambda: (self.search_edit.setFocus(), self.search_edit.selectAll()))
        self.shortcut("Ctrl+E", self.edit_current_prompt)
        self.shortcut("Delete", self.delete_current_prompt)
        self.shortcut("Ctrl+Shift+I", self.import_json)
        self.shortcut("Ctrl+Shift+E", self.export_json)

    def load_settings(self):
        try:
            if SETTINGS_FILE.exists():
                with SETTINGS_FILE.open("r", encoding="utf-8") as f:
                    data = json.load(f)
                if isinstance(data, dict):
                    return data
        except Exception:
            log_error()
        return {"theme": "深色", "custom_bg": "", "background_image": "", "category_icons": {}, "custom_categories": []}

    def save_settings(self):
        DATA_DIR.mkdir(parents=True, exist_ok=True)
        with SETTINGS_FILE.open("w", encoding="utf-8") as f:
            json.dump(self.settings, f, ensure_ascii=False, indent=2)

    def appearance_menu(self):
        menu = QMenu(self)
        for theme in ("深色", "浅色", "柔和绿"):
            action = menu.addAction(("✓ " if self.settings.get("theme", "深色") == theme else "   ") + theme)
            action.triggered.connect(lambda checked=False, t=theme: self.set_theme(t))
        menu.addSeparator()
        menu.addAction("自选背景颜色…", self.choose_background_color)
        menu.addAction("设置界面字体颜色…", self.choose_text_color)
        menu.addAction("设置提示词卡片字体颜色…", self.choose_card_text_color)
        menu.addAction("自动适配背景对比度", self.toggle_auto_contrast).setCheckable(True)
        menu.actions()[-1].setChecked(self.settings.get("auto_contrast", True))
        menu.addAction("字体粗细与大小…", self.configure_font)
        menu.addAction("提示词卡片宽度与高度…", self.configure_card_size)
        menu.addAction("恢复默认字体颜色", self.reset_text_colors)
        menu.addAction("上传背景图片…", self.choose_background_image)
        menu.addAction("清除自定义背景", self.clear_custom_background)
        menu.exec(self.appearance_btn.mapToGlobal(self.appearance_btn.rect().bottomLeft()))

    def toggle_auto_contrast(self, enabled):
        self.settings["auto_contrast"] = bool(enabled)
        if enabled:
            # 启用自动对比度时，移除手动界面文字颜色，让颜色随背景重新计算。
            self.settings.pop("text_color", None)
        self.save_settings()
        self.apply_theme()

    @staticmethod
    def _color_is_light(hex_color):
        """按相对亮度判断背景明暗，自动选择高对比度前景色。"""
        try:
            color = QColor(hex_color)
            if not color.isValid():
                return False
            def linear(v):
                v = v / 255.0
                return v / 12.92 if v <= 0.04045 else ((v + 0.055) / 1.055) ** 2.4
            luminance = 0.2126 * linear(color.red()) + 0.7152 * linear(color.green()) + 0.0722 * linear(color.blue())
            return luminance > 0.42
        except Exception:
            return False

    def set_theme(self, theme):
        self.settings["theme"] = theme
        self.settings["custom_bg"] = ""
        self.settings["background_image"] = ""
        self.save_settings()
        self.apply_theme()
        self.statusBar().showMessage(f"已切换到{theme}主题", 2000)

    def configure_font(self):
        """设置全局字体大小和三档粗细；数字框可用鼠标滚轮微调。"""
        dlg = QDialog(self)
        dlg.setWindowTitle("字体粗细与大小")
        dlg.setMinimumWidth(340)
        form = QFormLayout(dlg)

        weight_box = QComboBox()
        weight_box.addItems(["细 / 常规", "中等", "加粗"])
        weight_map = {"细 / 常规": 400, "中等": 500, "加粗": 700}
        current_weight = int(self.settings.get("font_weight", 400))
        weight_box.setCurrentText(next((k for k, v in weight_map.items() if v == current_weight), "细 / 常规"))

        family_box = QFontComboBox()
        family_box.setFontFilters(QFontComboBox.FontFilter.AllFonts)
        saved_family = self.settings.get("font_family", "")
        if saved_family:
            family_box.setCurrentFont(QFont(saved_family))

        size_box = QSpinBox()
        size_box.setRange(9, 24)
        size_box.setSuffix(" px")
        size_box.setValue(int(self.settings.get("font_size", 13)))
        size_box.setToolTip("可以点击上下箭头，或将鼠标放在数字上滚动滚轮调整大小")

        preview = QLabel("字体预览：PromptDesk 提示词管理器  中文文字  123")
        preview.setWordWrap(True)
        preview.setMinimumHeight(42)
        def update_preview(*_):
            f = preview.font()
            f.setFamily(family_box.currentFont().family())
            f.setPixelSize(size_box.value())
            f.setWeight(QFont.Weight(weight_map[weight_box.currentText()]))
            preview.setFont(f)
        size_box.valueChanged.connect(update_preview)
        weight_box.currentTextChanged.connect(update_preview)
        family_box.currentFontChanged.connect(update_preview)
        form.addRow("字体类型", family_box)
        form.addRow("字体粗细（三档）", weight_box)
        form.addRow("字体大小", size_box)
        form.addRow("实时预览", preview)
        buttons = QDialogButtonBox(QDialogButtonBox.StandardButton.Save | QDialogButtonBox.StandardButton.Cancel)
        buttons.accepted.connect(dlg.accept)
        buttons.rejected.connect(dlg.reject)
        form.addRow(buttons)
        update_preview()
        if dlg.exec() == QDialog.DialogCode.Accepted:
            self.settings["font_family"] = family_box.currentFont().family()
            self.settings["font_weight"] = weight_map[weight_box.currentText()]
            self.settings["font_size"] = size_box.value()
            self.save_settings()
            self.apply_theme()
            self.statusBar().showMessage("字体设置已保存", 2000)

    def configure_card_size(self):
        """允许用户单独调整提示词卡片的宽度和高度，并保存设置。"""
        dlg = QDialog(self)
        dlg.setWindowTitle("提示词卡片尺寸")
        dlg.setMinimumWidth(330)
        layout = QVBoxLayout(dlg)

        layout.addWidget(QLabel("卡片宽度（像素）"))
        width_box = QSpinBox()
        width_box.setRange(180, 600)
        width_box.setSingleStep(10)
        width_box.setSuffix(" px")
        width_box.setValue(int(self.settings.get("card_width", 326)))
        layout.addWidget(width_box)

        layout.addWidget(QLabel("卡片高度（像素）"))
        height_box = QSpinBox()
        height_box.setRange(90, 360)
        height_box.setSingleStep(10)
        height_box.setSuffix(" px")
        height_box.setValue(int(self.settings.get("card_height", 132)))
        layout.addWidget(height_box)

        layout.addWidget(QLabel("卡片之间的间隔（像素）"))
        spacing_box = QSpinBox()
        spacing_box.setRange(0, 50)
        spacing_box.setSingleStep(2)
        spacing_box.setSuffix(" px")
        spacing_box.setValue(int(self.settings.get("card_spacing", 10)))
        layout.addWidget(spacing_box)

        preview = QLabel("预览：分别调整卡片宽度、高度和卡片之间的间隔")
        preview.setWordWrap(True)
        layout.addWidget(preview)
        buttons = QDialogButtonBox(QDialogButtonBox.StandardButton.Ok | QDialogButtonBox.StandardButton.Cancel)
        buttons.accepted.connect(dlg.accept)
        buttons.rejected.connect(dlg.reject)
        layout.addWidget(buttons)

        if dlg.exec() == QDialog.DialogCode.Accepted:
            self.settings["card_width"] = width_box.value()
            self.settings["card_height"] = height_box.value()
            self.settings["card_spacing"] = spacing_box.value()
            self.prompt_list.setGridSize(QSize(width_box.value(), height_box.value()))
            self.prompt_list.setSpacing(spacing_box.value())
            self.prompt_list.doItemsLayout()
            self.prompt_list.viewport().update()
            self.save_settings()

    def choose_text_color(self):
        color = QColorDialog.getColor(QColor(self.settings.get("text_color") or "#e7edf3"), self, "选择界面字体颜色")
        if color.isValid():
            self.settings["text_color"] = color.name()
            self.save_settings()
            self.apply_theme()

    def choose_card_text_color(self):
        color = QColorDialog.getColor(QColor(self.settings.get("card_text_color") or "#e7edf3"), self, "选择提示词卡片字体颜色")
        if color.isValid():
            self.settings["card_text_color"] = color.name()
            self.save_settings()
            self.apply_theme()

    def reset_text_colors(self):
        self.settings.pop("text_color", None)
        self.settings.pop("card_text_color", None)
        self.save_settings()
        self.apply_theme()

    def choose_background_color(self):
        color = QColorDialog.getColor(QColor(self.settings.get("custom_bg") or "#101318"), self, "选择窗口背景颜色")
        if color.isValid():
            self.settings["custom_bg"] = color.name()
            self.settings["background_image"] = ""
            if self.settings.get("auto_contrast", True):
                self.settings.pop("text_color", None)
            self.save_settings()
            self.apply_theme()

    def choose_background_image(self):
        path, _ = QFileDialog.getOpenFileName(
            self, "选择背景图片", str(Path.home()),
            "图片文件 (*.png *.jpg *.jpeg *.bmp *.webp *.tif *.tiff)"
        )
        if not path:
            return
        try:
            DATA_DIR.mkdir(parents=True, exist_ok=True)
            optimized = DATA_DIR / "background_optimized.jpg"
            # 首选 Pillow 做轻度模糊、降饱和和暗色蒙层；没有 Pillow 时用 Qt 内置图像处理，
            # 避免因为缺少第三方组件而让上传背景功能完全失效。
            try:
                from PIL import Image, ImageEnhance, ImageFilter, ImageOps
                with Image.open(path) as source:
                    img = ImageOps.exif_transpose(source).convert("RGB")
                    img.thumbnail((1920, 1200), Image.Resampling.LANCZOS)
                    img = ImageEnhance.Color(img).enhance(0.72)
                    img = img.filter(ImageFilter.GaussianBlur(radius=1.0))
                    overlay = Image.new("RGB", img.size, (15, 19, 25))
                    img = Image.blend(img, overlay, 0.48)
                    img.save(str(optimized), "JPEG", quality=90, optimize=True)
            except ImportError:
                source_pixmap = QPixmap(path)
                if source_pixmap.isNull():
                    raise ValueError("Qt 无法读取这张图片。请换用常见的 PNG 或 JPG 图片。")
                pixmap = source_pixmap.scaled(1920, 1200, Qt.AspectRatioMode.KeepAspectRatio,
                                             Qt.TransformationMode.SmoothTransformation)
                # Qt 内置方式：压暗叠层确保浅色字体可读，不依赖 Pillow。
                painter = QPainter(pixmap)
                painter.fillRect(pixmap.rect(), QColor(15, 19, 25, 125))
                painter.end()
                if not pixmap.save(str(optimized), "JPG", 90):
                    raise OSError("优化后的背景图片无法保存，请检查 PromptDesk 数据目录权限。")

            check = QPixmap(str(optimized))
            if check.isNull() or check.width() < 1 or check.height() < 1:
                raise ValueError("背景图片处理完成后无法读取，请换一张图片再试。")
            self.settings["background_image"] = str(optimized.resolve())
            self.settings["background_original"] = str(Path(path).resolve())
            self.settings["custom_bg"] = ""
            # 背景压暗后使用浅色界面字体；卡片、输入框和列表保持不透明底色。
            self.settings["text_color"] = "#F5F7FA"
            self.settings["card_text_color"] = ""
            self.save_settings()
            self.apply_theme()
            self.statusBar().showMessage("背景已上传并自动优化；文字对比度已调整", 4000)
        except Exception as exc:
            log_error()
            QMessageBox.warning(self, "背景上传失败", f"无法应用这张背景图片：\n{exc}\n\n请尝试 PNG 或 JPG 图片。")

    def clear_custom_background(self):
        self.settings["custom_bg"] = ""
        self.settings["background_image"] = ""
        self.save_settings()
        self.apply_theme()

    def apply_theme(self):
        theme = self.settings.get("theme", "深色")
        palettes = {
            "深色": {"bg":"#101318", "panel":"#171c23", "text":"#e7edf3", "border":"#303945", "accent":"#83c64a", "button":"#202832", "menu":"#1b2129"},
            "浅色": {"bg":"#f2f5f8", "panel":"#ffffff", "text":"#202833", "border":"#d1d9e2", "accent":"#3478d4", "button":"#e6edf5", "menu":"#ffffff"},
            "柔和绿": {"bg":"#e8f0e8", "panel":"#f5faf3", "text":"#24352a", "border":"#c4d6c3", "accent":"#3c8b58", "button":"#dcebdd", "menu":"#f7fbf6"},
        }
        c = palettes.get(theme, palettes["深色"])
        custom = self.settings.get("custom_bg")
        image = self.settings.get("background_image", "")
        if custom:
            c["bg"] = custom
            # 自定义背景时，面板和控件同步调整明暗，避免出现白底白字或黑底黑字。
            if self._color_is_light(custom):
                c.update({"panel": "#FFFFFF", "button": "#F0F3F6", "menu": "#FFFFFF", "border": "#596575", "accent": "#174EA6", "text": "#111820"})
            else:
                c.update({"panel": "#20252D", "button": "#2B323C", "menu": "#20252D", "border": "#7C8998", "accent": "#B6F078", "text": "#FFFFFF"})
        if image and Path(image).is_file():
            # 背景图在上传时已压暗；所有界面文字统一采用高亮浅色。
            c.update({"text":":#F7F9FC".replace(":", ""), "panel":"#20252D", "button":"#2B323C", "menu":"#20252D", "border":"#66717F", "accent":"#B6F078"})
        if self.settings.get("auto_contrast", True):
            if custom:
                c["text"] = "#111820" if self._color_is_light(custom) else "#FFFFFF"
            elif image and Path(image).is_file():
                c["text"] = "#F7F9FC"
        elif self.settings.get("text_color"):
            c["text"] = self.settings["text_color"]
        font_size = max(9, min(24, int(self.settings.get("font_size", 13))))
        font_weight = max(400, min(700, int(self.settings.get("font_weight", 400))))
        font_family = self.settings.get("font_family", "")
        try:
            app_font = QApplication.font()
            app_font.setPixelSize(font_size)
            if font_family:
                app_font.setFamily(self.settings.get("font_family", ""))
            app_font.setWeight(QFont.Weight(font_weight))
            QApplication.setFont(app_font)
        except Exception:
            pass
        # 背景由 BackgroundWidget 自行等比绘制并居中裁切，窗口全屏铺满但不会拉伸变形。
        if hasattr(self, "central_widget"):
            self.central_widget.set_background(image if image and Path(image).is_file() else "", c["bg"])
        central_rule = 'QWidget#centralBackground { background-color: transparent; }'
        self.setStyleSheet(f"""
            QMainWindow {{ background-color: {c['bg']}; color: {c['text']}; font-family: "{font_family}"; font-size: {font_size}px; font-weight: {font_weight}; }}
            QWidget {{ color: {c['text']}; font-family: "{font_family}"; font-size: {font_size}px; font-weight: {font_weight}; background-color: transparent; }}
            {central_rule}
            QLabel#brand {{ color: {c['accent']}; font-size: 21px; font-weight: 800; letter-spacing: 2px; background: transparent; }}
            QLabel#muted {{ color: {c['text']}; background: transparent; }}
            QListWidget {{ background-color: {c['panel']}; color: {c['text']}; border: 2px solid {c['border']}; border-radius: 10px; padding: 8px; outline: 0; }}
            QLineEdit, QTextEdit, QComboBox, QSpinBox {{ background-color: {c['panel']}; color: {c['text']}; border: 2px solid {c['border']}; border-radius: 7px; padding: 9px; selection-background-color: {c['accent']}; }}
            QComboBox QAbstractItemView, QListWidget, QTreeWidget, QTableWidget {{ background-color: {c['panel']}; color: {c['text']}; selection-background-color: {c['accent']}; selection-color: {c['text']}; }}
            QCheckBox, QRadioButton, QGroupBox, QTabWidget, QTabBar, QToolTip {{ color: {c['text']}; }}
            QPushButton, QToolButton {{ background-color: {c['button']}; color: {c['text']}; border: 2px solid {c['border']}; border-radius: 7px; padding: 8px 12px; }}
            QPushButton:hover, QToolButton:hover {{ border-color: {c['accent']}; }}
            QLineEdit:focus, QTextEdit:focus, QComboBox:focus {{ border: 2px solid {c['accent']}; }}
            QListWidget:focus {{ border: 2px solid {c['accent']}; }}
            QMenu {{ background-color: {c['menu']}; color: {c['text']}; border: 1px solid {c['border']}; padding: 5px; }}
            QMenu::item {{ padding: 7px 25px 7px 12px; border-radius: 4px; }}
            QMenu::item:selected {{ background-color: {c['button']}; }}
            QStatusBar {{ color: {c['text']}; background-color: {c['panel']}; border-top: 1px solid {c['border']}; }}
            QToolTip {{ background-color: {c['panel']}; color: {c['text']}; border: 1px solid {c['border']}; padding: 5px; }}
            QScrollBar:vertical {{ background: {c['panel']}; width: 12px; margin: 2px; }}
            QScrollBar::handle:vertical {{ background: {c['border']}; min-height: 24px; border-radius: 5px; }}
            QScrollBar:horizontal {{ background: {c['panel']}; height: 12px; margin: 2px; }}
            QScrollBar::handle:horizontal {{ background: {c['border']}; min-width: 24px; border-radius: 5px; }}
            QSplitter::handle {{ background-color: {c['border']}; width: 5px; }}
        """)
        if hasattr(self, "card_delegate"):
            self.card_delegate.theme = theme
            self.card_delegate.custom_text_color = self.settings.get("card_text_color", "")
            self.card_delegate.prompt_font_size = font_size
            self.card_delegate.prompt_font_weight = QFont.Weight(font_weight)
            self.card_delegate.prompt_font_family = self.settings.get("font_family", "")
            self.prompt_list.viewport().update()

    def set_category_icon(self, category):
        # 可视化预设图标选择器：分类图标不再只能从右键菜单中选文字。
        choices = [
            "📁", "🗂️", "📂", "📝", "✍️", "📚", "📖", "🎨", "🖼️", "📷", "🎬", "🎥",
            "🎞️", "🎵", "🎧", "🌐", "🔤", "💬", "💼", "📊", "📌", "⭐", "🌟", "🔥",
            "💡", "🤖", "🧠", "⚙️", "🔧", "🧩", "🌿", "🌈", "✨", "❤️", "🙏", "✝️",
            "⛪", "🕊️", "👨‍👩‍👧", "🏷️", "✅", "🚀", "💎", "🎯", "🔔", "📦", "🛠️", "🌍"
        ]
        dlg = QDialog(self)
        dlg.setWindowTitle(f"选择「{category}」的图标")
        dlg.resize(440, 390)
        outer = QVBoxLayout(dlg)
        outer.addWidget(QLabel("选择一个预设图标，也可以上传自己的图标图片："))
        grid_widget = QWidget()
        grid = QGridLayout(grid_widget)
        grid.setSpacing(6)
        for i, emoji in enumerate(choices):
            btn = QToolButton()
            btn.setText(emoji)
            btn.setToolTip(emoji)
            btn.setFont(QFont("Segoe UI Emoji", 19))
            btn.setFixedSize(48, 43)
            btn.clicked.connect(lambda checked=False, e=emoji: (self._save_category_emoji(category, e), dlg.accept()))
            grid.addWidget(btn, i // 8, i % 8)
        outer.addWidget(grid_widget, 1)
        actions = QHBoxLayout()
        upload = QPushButton("上传自定义图标…")
        upload.clicked.connect(lambda: (self.upload_category_icon(category), dlg.accept()))
        reset = QPushButton("恢复默认")
        reset.clicked.connect(lambda: (self._remove_category_icon(category), dlg.accept()))
        cancel = QPushButton("取消")
        cancel.clicked.connect(dlg.reject)
        actions.addWidget(upload)
        actions.addWidget(reset)
        actions.addStretch(1)
        actions.addWidget(cancel)
        outer.addLayout(actions)
        dlg.exec()

    def _save_category_emoji(self, category, emoji):
        self.settings.setdefault("category_icons", {})[category] = "emoji:" + emoji
        self.save_settings()
        self.refresh_categories()

    def upload_category_icon(self, category):
        path, _ = QFileDialog.getOpenFileName(self, "选择分类图标", str(Path.home()), "图片文件 (*.png *.jpg *.jpeg *.bmp *.webp *.ico)")
        if path:
            self.settings.setdefault("category_icons", {})[category] = path
            self.save_settings()
            self.refresh_categories()

    def _remove_category_icon(self, category):
        self.settings.setdefault("category_icons", {}).pop(category, None)
        self.save_settings()
        self.refresh_categories()

    def shortcut(self, key, callback):
        action = QAction(self)
        action.setShortcut(QKeySequence(key))
        action.triggered.connect(callback)
        self.addAction(action)

    def load_hidden_categories(self):
        try:
            if HIDDEN_CATEGORIES_FILE.exists():
                with HIDDEN_CATEGORIES_FILE.open("r", encoding="utf-8") as f:
                    data = json.load(f)
                if isinstance(data, list):
                    self.hidden_categories = {str(x) for x in data if str(x) not in ("全部提示词", "未分类", "⭐ 收藏", "📌 置顶", "🗑 回收站")}
        except Exception:
            log_error()
            self.hidden_categories = set()

    def save_hidden_categories(self):
        DATA_DIR.mkdir(parents=True, exist_ok=True)
        with HIDDEN_CATEGORIES_FILE.open("w", encoding="utf-8") as f:
            json.dump(sorted(self.hidden_categories), f, ensure_ascii=False, indent=2)

    def load_data(self):
        if not DATA_FILE.exists():
            # 首次启动时添加一些示例提示词，方便用户直观看到卡片布局和使用方式。
            self.items = [
                new_prompt("写作｜专业润色", "请将以下内容润色得清晰、自然、专业，保留原意，不要添加未经提供的事实：\n\n【粘贴原文】", "写作"),
                new_prompt("写作｜提炼重点", "请阅读以下内容，提炼 5 个核心要点，并用简洁的项目符号呈现：\n\n【粘贴内容】", "写作"),
                new_prompt("图片｜写实场景", "生成一张超写实、电影级画面的图片。主体清晰，构图自然，光线真实，细节丰富，无文字、无水印。场景：【填写场景】。", "图片生成"),
                new_prompt("图片｜竖屏 9:16", "9:16 竖屏构图，真实摄影质感，自然光照，主体完整清晰，背景具有层次感，色彩协调，无文字、无水印。内容：【填写主题】。", "图片生成"),
                new_prompt("视频｜电影感镜头", "制作一段电影感写实视频。镜头运动平稳，人物动作自然连贯，环境细节真实，光影统一，不出现字幕或水印。时长：【填写秒数】；场景：【填写场景】。", "视频生成"),
                new_prompt("翻译｜自然地道", "请将以下文字翻译成自然、地道的【目标语言】，准确保留原意和语气，不增删信息，只输出译文：\n\n【粘贴原文】", "翻译"),
                new_prompt("工作｜拆解任务", "请把以下目标拆解为可执行步骤，并列出每一步的具体操作、所需工具、预计耗时和完成标准：\n\n【填写目标】", "工作"),
                new_prompt("通用｜优化提示词", "请优化下面的提示词，使其目标明确、上下文充分、输出要求具体，并补充必要的限制条件。先给出优化后的版本，再简要说明关键改动：\n\n【粘贴原提示词】", "未分类"),
            ]
            self.save_data()
            return
        try:
            with DATA_FILE.open("r", encoding="utf-8") as f:
                data = json.load(f)
            self.items = data if isinstance(data, list) else []
            for p in self.items:
                p.setdefault("id", str(uuid.uuid4()))
                p.setdefault("title", "未命名提示词")
                p.setdefault("content", "")
                p.setdefault("category", "未分类")
                p.setdefault("favorite", False)
                p.setdefault("pinned", False)
                p.setdefault("deleted", False)
                p.setdefault("created", datetime.now().isoformat(timespec="seconds"))
                p.setdefault("updated", p["created"])
                p.setdefault("used", "")
        except Exception:
            log_error()
            QMessageBox.warning(self, "读取数据失败", f"无法读取提示词数据。程序已记录错误日志：\n{LOG_FILE}\n\n将以空列表启动，不会自动覆盖原文件。")
            self.items = []

    def save_data(self):
        DATA_DIR.mkdir(parents=True, exist_ok=True)
        temp = DATA_FILE.with_suffix(".tmp")
        with temp.open("w", encoding="utf-8") as f:
            json.dump(self.items, f, ensure_ascii=False, indent=2)
        temp.replace(DATA_FILE)

    def categories(self):
        # 同时读取设置中保存的自定义分类，这样新建但暂时没有提示词的分类也不会消失。
        prompt_categories = {p.get("category", "未分类") for p in self.items if p.get("category")}
        saved_categories = self.settings.get("custom_categories", []) if isinstance(self.settings, dict) else []
        custom = sorted({str(c).strip() for c in list(prompt_categories) + list(saved_categories)
                         if str(c).strip() and str(c) not in ("全部提示词", "未分类")})
        result = []
        for c in DEFAULT_CATEGORIES:
            if c not in result and c not in self.hidden_categories:
                result.append(c)
        for c in custom:
            if c not in result and c not in self.hidden_categories:
                result.append(c)
        return result

    def refresh_categories(self):
        old = self.current_category
        self._loading = True
        self.category_list.clear()
        counts = {c: sum(1 for p in self.items if not p.get("deleted") and p.get("category") == c) for c in self.categories()}
        active_count = sum(1 for p in self.items if not p.get("deleted"))
        trash_count = sum(1 for p in self.items if p.get("deleted"))
        entries = [("全部提示词", active_count), ("⭐ 收藏", -1), ("🗑 回收站", trash_count)]
        for c in self.categories():
            if c != "全部提示词":
                entries.append((c, counts.get(c, 0)))
        target = None
        for name, count in entries:
            label = name if count < 0 else f"{name}    {count}"
            item = QListWidgetItem(label)
            item.setData(Qt.ItemDataRole.UserRole, name)
            icon_value = self.settings.get("category_icons", {}).get(name)
            if icon_value:
                if icon_value.startswith("emoji:"):
                    item.setText(icon_value[6:] + "  " + label)
                elif Path(icon_value).exists():
                    item.setIcon(QIcon(icon_value))
            self.category_list.addItem(item)
            if name == old:
                target = item
        if target is None:
            target = self.category_list.item(0)
            self.current_category = "全部提示词"
        if target:
            self.category_list.setCurrentItem(target)
        self._loading = False

    def toggle_category_panel(self, checked=None):
        """使用左上角的三点按钮显示或隐藏左侧分类栏。"""
        if not hasattr(self, "left_panel"):
            return
        visible = self.category_toggle_btn.isChecked()
        self.left_panel.setVisible(visible)
        if visible:
            self.category_toggle_btn.setText("◧")
            self.category_toggle_btn.setToolTip("隐藏左侧分类栏（右键打开分类管理）")
            # 恢复到合适的侧栏宽度，避免重新显示时挤得太窄。
            sizes = self.main_splitter.sizes()
            if len(sizes) >= 2 and sizes[0] < 120:
                total = max(sum(sizes), self.width())
                left = max(190, int(total * 0.22))
                self.main_splitter.setSizes([left, max(300, total - left)])
        else:
            self.category_toggle_btn.setText("▤")
            self.category_toggle_btn.setToolTip("显示左侧分类栏（右键打开分类管理）")

    def on_category_changed(self, current, previous=None):
        if self._loading or current is None:
            return
        self.current_category = current.data(Qt.ItemDataRole.UserRole)
        self.refresh_list()

    def visible_items(self):
        # 顶部搜索为全库搜索：输入关键词时，不受当前分类限制。
        # 支持搜索标题、提示词正文、分类名称和备注/标注；清空关键词后恢复当前分类视图。
        query = self.search_edit.text().strip().casefold()
        category = self.current_category
        result = []
        for p in self.items:
            deleted = bool(p.get("deleted"))
            if query:
                # 搜索时仍区分正常列表与回收站，避免已删除内容混入正常结果。
                if category == "🗑 回收站":
                    if not deleted:
                        continue
                elif deleted:
                    continue
                searchable = " ".join(str(p.get(key, "") or "") for key in ("title", "content", "category", "annotation")).casefold()
                if query not in searchable:
                    continue
            else:
                if category == "🗑 回收站":
                    if not deleted:
                        continue
                else:
                    if deleted:
                        continue
                    if category == "⭐ 收藏" and not p.get("favorite"):
                        continue
                    elif category not in ("全部提示词", "⭐ 收藏") and p.get("category", "未分类") != category:
                        continue
            result.append(p)
        result.sort(key=lambda p: (not bool(p.get("pinned")), not bool(p.get("favorite")), p.get("title", "").casefold()))
        return result

    def refresh_list(self):
        # Refreshing/searching never copies content; only a deliberate click does.
        self._loading = True
        self.prompt_list.clear()
        items = self.visible_items()
        for p in items:
            flags = ("📌 " if p.get("pinned") else "") + ("⭐ " if p.get("favorite") else "") + ("🏷️ " if p.get("annotation") else "")
            item = QListWidgetItem(flags)
            item.setData(Qt.ItemDataRole.UserRole, p["id"])
            item.setData(Qt.ItemDataRole.UserRole + 1, p.get("content", ""))
            item.setData(Qt.ItemDataRole.UserRole + 2, p.get("category", "未分类"))
            item.setToolTip(f"分类：{p.get('category', '未分类')}\n标注：{p.get('annotation', '') or '无'}\n提示词内容：\n{p.get('content', '')}\n单击复制内容 · 双击编辑")
            self.prompt_list.addItem(item)
        self.count_label.setText(f"{len(items)} 条")
        self._loading = False

    def get_prompt(self, prompt_id):
        return next((p for p in self.items if p.get("id") == prompt_id), None)

    def current_prompt(self):
        item = self.prompt_list.currentItem()
        return self.get_prompt(item.data(Qt.ItemDataRole.UserRole)) if item else None

    def on_prompt_clicked(self, item):
        if self._loading or not item:
            return
        if getattr(self.prompt_list, "drag_selecting", False):
            return
        p = self.get_prompt(item.data(Qt.ItemDataRole.UserRole))
        if not p:
            return
        if p.get("deleted"):
            self.statusBar().showMessage("回收站中的提示词不会自动复制", 2500)
            return
        content = p.get("content", "")
        QApplication.clipboard().setText(content)
        # 主动点击即复制；空内容也给出明确反馈，方便发现提示词是否为空。
        p["used"] = datetime.now().isoformat(timespec="seconds")
        try:
            self.save_data()
        except Exception:
            log_error()
        if content:
            self.statusBar().showMessage(f"已复制提示词内容 · 分类：{p.get('category', '未分类')}", 2500)
        else:
            self.statusBar().showMessage("该提示词内容为空，已复制空内容", 3000)

    def create_prompt(self):
        dlg = PromptDialog(self, categories=self.categories())
        if dlg.exec() != QDialog.DialogCode.Accepted:
            return
        title, content, category, annotation = dlg.values()
        p = new_prompt(title, content, category)
        p["annotation"] = annotation
        self.items.append(p)
        self.current_category = category
        self.save_data()
        self.refresh_categories()
        self.refresh_list()
        self.statusBar().showMessage(f"已创建：{title}", 2500)

    def edit_prompt_item(self, item=None):
        p = self.get_prompt(item.data(Qt.ItemDataRole.UserRole)) if item else self.current_prompt()
        if not p or p.get("deleted"):
            return
        dlg = PromptDialog(self, p, self.categories())
        if dlg.exec() != QDialog.DialogCode.Accepted:
            return
        title, content, category, annotation = dlg.values()
        p.update({"title": title, "content": content, "category": category, "annotation": annotation, "updated": datetime.now().isoformat(timespec="seconds")})
        self.current_category = category
        self.save_data()
        self.refresh_categories()
        self.refresh_list()
        self.statusBar().showMessage("已保存修改", 2500)

    def edit_current_prompt(self):
        self.edit_prompt_item()

    def delete_current_prompt(self):
        p = self.current_prompt()
        if p:
            self.delete_prompt(p)

    def delete_selected_prompts(self):
        selected = self.prompt_list.selectedItems()
        prompts = [self.get_prompt(item.data(Qt.ItemDataRole.UserRole)) for item in selected]
        prompts = [p for p in prompts if p]
        if not prompts:
            QMessageBox.information(self, "批量删除", "请先按住鼠标左键拖动框选要删除的提示词卡片。也可以按住 Ctrl 再逐个点击选择。")
            return
        in_trash = all(bool(p.get("deleted")) for p in prompts)
        if in_trash:
            answer = QMessageBox.question(self, "批量永久删除", f"确定永久删除选中的 {len(prompts)} 条提示词吗？此操作无法撤销。")
            if answer != QMessageBox.StandardButton.Yes:
                return
            ids = {p.get("id") for p in prompts}
            self.items = [p for p in self.items if p.get("id") not in ids]
            message = f"已永久删除 {len(prompts)} 条提示词"
        else:
            answer = QMessageBox.question(self, "批量移入回收站", f"确定将选中的 {len(prompts)} 条提示词移入回收站吗？")
            if answer != QMessageBox.StandardButton.Yes:
                return
            for p in prompts:
                p["deleted"] = True
            message = f"已将 {len(prompts)} 条提示词移入回收站"
        self.save_data()
        self.refresh_categories()
        self.refresh_list()
        self.statusBar().showMessage(message, 3000)

    def delete_prompt(self, p):
        if p.get("deleted"):
            answer = QMessageBox.question(self, "永久删除", f"确定永久删除“{p.get('title')}”吗？此操作无法撤销。")
            if answer != QMessageBox.StandardButton.Yes:
                return
            self.items = [x for x in self.items if x.get("id") != p.get("id")]
        else:
            answer = QMessageBox.question(self, "移入回收站", f"将“{p.get('title')}”移入回收站？")
            if answer != QMessageBox.StandardButton.Yes:
                return
            p["deleted"] = True
        self.save_data()
        self.refresh_categories()
        self.refresh_list()
        self.statusBar().showMessage("操作完成", 2500)

    def restore_prompt(self, p):
        p["deleted"] = False
        self.save_data()
        self.refresh_categories()
        self.refresh_list()
        self.statusBar().showMessage("已恢复提示词", 2500)

    def toggle_flag(self, p, flag, label):
        p[flag] = not bool(p.get(flag))
        p["updated"] = datetime.now().isoformat(timespec="seconds")
        self.save_data()
        self.refresh_categories()
        self.refresh_list()
        self.statusBar().showMessage(f"已{'' if p[flag] else '取消'}{label}", 2500)

    def copy_content(self, p):
        QApplication.clipboard().setText(p.get("content", ""))
        self.statusBar().showMessage(f"已复制提示词内容 · 分类：{p.get('category', '未分类')}", 2500)

    def rename_prompt(self, p):
        title, ok = QInputDialog.getText(self, "重命名", "提示词标题：", QLineEdit.EchoMode.Normal, p.get("title", ""))
        if ok and title.strip():
            p["title"] = title.strip()
            p["updated"] = datetime.now().isoformat(timespec="seconds")
            self.save_data(); self.refresh_list()

    def move_prompt(self, p):
        cats = ["未分类"] + [c for c in self.categories() if c not in ("全部提示词", "未分类")]
        category, ok = QInputDialog.getItem(self, "移动到分类", "选择目标分类：", cats, max(0, cats.index(p.get("category")) if p.get("category") in cats else 0), False)
        if ok and category:
            p["category"] = category
            p["updated"] = datetime.now().isoformat(timespec="seconds")
            self.save_data(); self.refresh_categories(); self.refresh_list()

    def prompt_context_menu(self, pos):
        item = self.prompt_list.itemAt(pos)
        menu = QMenu(self)
        if item:
            if len(self.prompt_list.selectedItems()) > 1:
                self.add_menu_action(menu, f"批量删除选中的 {len(self.prompt_list.selectedItems())} 条…", self.delete_selected_prompts)
                menu.addSeparator()
            p = self.get_prompt(item.data(Qt.ItemDataRole.UserRole))
            if not p: return
            if p.get("deleted"):
                self.add_menu_action(menu, "恢复提示词", lambda: self.restore_prompt(p))
                self.add_menu_action(menu, "永久删除", lambda: self.delete_prompt(p))
            else:
                self.add_menu_action(menu, "复制提示词内容", lambda: self.copy_content(p))
                menu.addSeparator()
                self.add_menu_action(menu, "编辑", lambda: self.edit_prompt_item(item))
                self.add_menu_action(menu, "重命名", lambda: self.rename_prompt(p))
                self.add_menu_action(menu, "移动到其他分类…", lambda: self.move_prompt(p))
                menu.addSeparator()
                self.add_menu_action(menu, "取消置顶" if p.get("pinned") else "置顶", lambda: self.toggle_flag(p, "pinned", "置顶"))
                self.add_menu_action(menu, "取消收藏" if p.get("favorite") else "收藏", lambda: self.toggle_flag(p, "favorite", "收藏"))
                menu.addSeparator()
                self.add_menu_action(menu, "移入回收站", lambda: self.delete_prompt(p))
        else:
            self.add_menu_action(menu, "新建提示词", self.create_prompt)
            self.add_menu_action(menu, "导入提示词文件…", self.import_json)
            self.add_menu_action(menu, "导出全部提示词…", self.export_json)
            self.add_menu_action(menu, "创建备份", self.create_backup)
            self.add_menu_action(menu, "打开数据文件夹", self.open_data_folder)
        menu.exec(self.prompt_list.mapToGlobal(pos))

    def add_menu_action(self, menu, text, callback):
        action = menu.addAction(text)
        action.triggered.connect(callback)
        return action

    def category_context_menu(self, pos):
        item = self.category_list.itemAt(pos)
        menu = QMenu(self)
        if item:
            cat = item.data(Qt.ItemDataRole.UserRole)
            if cat not in ("全部提示词", "⭐ 收藏", "📌 置顶", "🗑 回收站"):
                self.add_menu_action(menu, "新建提示词", lambda: self.create_prompt())
                self.add_menu_action(menu, "设置分类图标…", lambda: self.set_category_icon(cat))
                self.add_menu_action(menu, "重命名分类", lambda: self.rename_category(cat))
                self.add_menu_action(menu, "删除分类", lambda: self.delete_category(cat))
                menu.addSeparator()
                self.add_menu_action(menu, "导出本分类提示词…", lambda: self.export_json(category=cat))
            else:
                self.add_menu_action(menu, "新建提示词", self.create_prompt)
        self.add_menu_action(menu, "新建分类", self.create_category)
        self.add_menu_action(menu, "导入提示词文件…", self.import_json)
        self.add_menu_action(menu, "导出全部提示词…", self.export_json)
        self.add_menu_action(menu, "创建备份", self.create_backup)
        menu.exec(self.category_list.mapToGlobal(pos))

    def category_menu(self, global_pos):
        menu = QMenu(self)
        self.add_menu_action(menu, "新建分类", self.create_category)
        self.add_menu_action(menu, "导入提示词文件…", self.import_json)
        self.add_menu_action(menu, "导出全部提示词…", self.export_json)
        self.add_menu_action(menu, "创建备份", self.create_backup)
        menu.exec(global_pos)

    def create_category(self):
        name, ok = QInputDialog.getText(self, "新建分类", "分类名称：")
        name = name.strip()
        if not ok or not name: return
        if name in ("全部提示词", "⭐ 收藏", "📌 置顶", "🗑 回收站") or name in self.categories():
            QMessageBox.warning(self, "分类已存在", "该分类名称不可用或已经存在。")
            return
        self.hidden_categories.discard(name)
        self.save_hidden_categories()
        custom_categories = self.settings.setdefault("custom_categories", [])
        if name not in custom_categories:
            custom_categories.append(name)
        self.save_settings()
        self.current_category = name
        self.refresh_categories()
        # 选择刚创建的分类；分类为空时也应保持可见。
        for i in range(self.category_list.count()):
            item = self.category_list.item(i)
            if item.data(Qt.ItemDataRole.UserRole) == name:
                self.category_list.setCurrentItem(item)
                break
        self.refresh_list()
        self.statusBar().showMessage(f"已新建分类：{name}", 2500)

    def rename_category(self, old):
        name, ok = QInputDialog.getText(self, "重命名分类", "新分类名称：", QLineEdit.EchoMode.Normal, old)
        name = name.strip()
        if not ok or not name or name == old: return
        if name in self.categories() or name in ("全部提示词", "⭐ 收藏", "📌 置顶", "🗑 回收站"):
            QMessageBox.warning(self, "名称不可用", "该分类名称已存在或不可使用。")
            return
        for p in self.items:
            if p.get("category") == old: p["category"] = name
        custom_categories = self.settings.setdefault("custom_categories", [])
        if old in custom_categories:
            custom_categories.remove(old)
        if name not in custom_categories:
            custom_categories.append(name)
        icons = self.settings.setdefault("category_icons", {})
        if old in icons: icons[name] = icons.pop(old)
        self.save_settings()
        if self.current_category == old: self.current_category = name
        self.save_data(); self.refresh_categories(); self.refresh_list()

    def delete_category(self, cat):
        if cat in ("全部提示词", "未分类", "⭐ 收藏", "📌 置顶", "🗑 回收站"):
            QMessageBox.information(self, "无法删除", "“全部提示词”“未分类”“收藏”“置顶”和“回收站”是系统分类，不能删除。")
            return
        if any(p.get("category") == cat for p in self.items):
            answer = QMessageBox.question(self, "删除分类", f"确定删除分类“{cat}”吗？该分类中的提示词会移到“未分类”。")
            if answer != QMessageBox.StandardButton.Yes: return
            for p in self.items:
                if p.get("category") == cat: p["category"] = "未分类"
        # 隐藏默认分类，避免刷新后重新出现；也支持删除自定义分类。
        self.hidden_categories.add(cat)
        self.save_hidden_categories()
        custom_categories = self.settings.setdefault("custom_categories", [])
        if cat in custom_categories:
            custom_categories.remove(cat)
        self.settings.setdefault("category_icons", {}).pop(cat, None)
        self.save_settings()
        self.current_category = "全部提示词"
        self.save_data(); self.refresh_categories(); self.refresh_list()
        self.statusBar().showMessage(f"已删除分类：{cat}", 2500)

    def import_json(self):
        path, _ = QFileDialog.getOpenFileName(self, "导入提示词", str(Path.home()), "JSON 文件 (*.json)")
        if not path: return
        try:
            with open(path, "r", encoding="utf-8-sig") as f: data = json.load(f)
            if isinstance(data, dict): data = data.get("items", data.get("prompts", []))
            if not isinstance(data, list): raise ValueError("JSON 内容应为提示词数组")
            existing = {p.get("id") for p in self.items}
            added = 0
            for raw in data:
                if not isinstance(raw, dict): continue
                title = str(raw.get("title", raw.get("name", "导入的提示词")))
                content = str(raw.get("content", raw.get("prompt", "")))
                category = str(raw.get("category", "未分类"))
                p = new_prompt(title, content, category)
                if raw.get("id") and raw.get("id") not in existing: p["id"] = str(raw["id"])
                p["favorite"] = bool(raw.get("favorite", False)); p["pinned"] = bool(raw.get("pinned", False))
                p["deleted"] = False
                self.items.append(p); existing.add(p["id"]); added += 1
            self.save_data(); self.refresh_categories(); self.refresh_list()
            QMessageBox.information(self, "导入完成", f"已导入 {added} 条提示词。")
        except Exception as e:
            log_error(); QMessageBox.critical(self, "导入失败", f"无法导入文件：\n{e}\n\n错误日志：{LOG_FILE}")

    def export_json(self, category=None):
        rows = [p for p in self.items if not p.get("deleted") and (category is None or p.get("category") == category)]
        default = f"prompts_{datetime.now():%Y%m%d_%H%M%S}.json"
        path, _ = QFileDialog.getSaveFileName(self, "导出提示词", str(Path.home() / default), "JSON 文件 (*.json)")
        if not path: return
        try:
            with open(path, "w", encoding="utf-8") as f: json.dump(rows, f, ensure_ascii=False, indent=2)
            self.statusBar().showMessage(f"已导出 {len(rows)} 条提示词", 3000)
        except Exception as e:
            log_error(); QMessageBox.critical(self, "导出失败", str(e))

    def create_backup(self):
        try:
            stamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            target = BACKUP_DIR / f"prompts_{stamp}.json"
            shutil.copy2(DATA_FILE, target)
            self.statusBar().showMessage(f"备份已保存：{target}", 4000)
            QMessageBox.information(self, "备份完成", f"备份文件：\n{target}")
        except Exception as e:
            log_error(); QMessageBox.critical(self, "备份失败", str(e))

    def open_data_folder(self):
        try:
            if sys.platform == "win32": os.startfile(str(DATA_DIR))
            else:
                import subprocess
                subprocess.Popen(["open" if sys.platform == "darwin" else "xdg-open", str(DATA_DIR)])
        except Exception as e:
            QMessageBox.warning(self, "无法打开", f"数据目录：\n{DATA_DIR}\n\n{e}")

    def closeEvent(self, event):
        try: self.save_data()
        except Exception: log_error()
        event.accept()


def main():
    app = QApplication(sys.argv)
    app.setApplicationName(APP_NAME)
    app.setFont(QFont("Microsoft YaHei UI", 10))
    try:
        window = PromptDesk()
        window.show()
        sys.exit(app.exec())
    except Exception:
        log_error()
        try:
            QMessageBox.critical(None, "PromptDesk 启动失败", f"程序遇到错误。错误日志位置：\n{LOG_FILE}\n\n请打开日志文件查看详情。")
        except Exception:
            pass
        raise

if __name__ == "__main__":
    main()
