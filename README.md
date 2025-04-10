import sys
import re
import os
import logging
from itertools import product
from PyQt6.QtWidgets import (
    QApplication, QWidget, QPushButton, QVBoxLayout, QFileDialog, QTextEdit,
    QHBoxLayout, QGridLayout, QLineEdit, QLabel, QScrollArea, QMessageBox, QDialog,
    QListWidget, QInputDialog, QSizePolicy, QFrame
)
from PyQt6.QtCore import Qt, QRect
from PyQt6.QtGui import QFont, QColor, QPalette, QTextCursor, QIntValidator, QWheelEvent

# 配置日志
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('app.log'),
        logging.StreamHandler(sys.stdout)
    ]
)

class DataParser:
    @staticmethod
    def parse_line(line):
        """解析单行彩票数据"""
        line = line.strip()
        date_period_match = re.search(r'(\d{8})\s*(\d{3})', line)
        if date_period_match:
            date_period = f"{date_period_match.group(1)}{date_period_match.group(2)}"
        else:
            date_period = "未知日期期数"

        numbers = []
        parts = line.split()
        if len(parts) >= 3:
            last_part = parts[-1]
            if last_part.isdigit() and len(last_part) == 4:
                numbers = list(last_part)

        if not numbers:
            all_digits = re.findall(r'\d', line)
            if len(all_digits) >= 5:
                numbers = all_digits[-5:-1]
            elif len(all_digits) >= 4:
                numbers = all_digits[-4:]

        return date_period, numbers[:4]

    @staticmethod
    def format_output(date_period, numbers):
        """格式化输出"""
        if len(numbers) != 4:
            return None

        date_period_str = f"{date_period[:8]}{date_period[8:11]}".ljust(12)
        size = "".join(["大" if int(n) >= 5 else "小" for n in numbers])
        parity = "".join(["单" if int(n) % 2 else "双" for n in numbers])
        return f"{date_period_str}  {''.join(numbers)}  {size}  {parity}"

class MainApp(QWidget):
    def __init__(self):
        super().__init__()
        try:
            self.init_ui()
        except Exception as e:
            self.show_error(f"初始化失败: {str(e)}")
            logging.exception("初始化失败")

    def show_error(self, message):
        msg = QMessageBox()
        msg.setIcon(QMessageBox.Icon.Critical)
        msg.setText("程序错误")
        msg.setInformativeText(message)
        msg.setWindowTitle("错误")
        msg.exec()

    def init_ui(self):
        screen = QApplication.primaryScreen().geometry()
        self.setGeometry(100, 100, int(screen.width()*0.15), int(screen.height()*0.15))
        self.setWindowTitle("彩票分析系统")
        self.setStyleSheet("""
            background-color: #E53935; 
            border: 5px solid #E53935;
            border-radius: 10px;
        """)

        layout = QVBoxLayout()
        buttons = ["彩票分析", "娱乐模式", "工具设置"]
        for name in buttons:
            btn = QPushButton(name)
            btn.setFixedHeight(50)
            btn.setStyleSheet("""
                QPushButton {
                    color: white; font-size: 18px; 
                    font-weight: bold;
                    background-color: #E53935;
                    border: 2px solid white;
                    border-radius: 8px;
                    margin: 1px;
                }
                QPushButton:hover { background-color: #EF5350; }
                QPushButton:pressed { background-color: #C62828; }
            """)
            if name == "彩票分析":
                btn.clicked.connect(self.open_lottery_window)
            layout.addWidget(btn)
        self.setLayout(layout)

    def open_lottery_window(self):
        try:
            self.lottery_window = LotteryWindow()
            self.lottery_window.show()
        except Exception as e:
            self.show_error(f"打开彩票窗口失败: {str(e)}")
            logging.exception("打开彩票窗口失败")

class LotteryWindow(QWidget):
    def __init__(self):
        super().__init__()
        try:
            self.parser = DataParser()
            self.data = []
            self.archive_files = {}
            self.init_ui()
            self.load_archive_files()
        except Exception as e:
            QMessageBox.critical(self, "错误", f"初始化失败: {str(e)}")
            logging.exception("彩票窗口初始化失败")

    def init_ui(self):
        self.setWindowTitle("彩票数据分析")
        screen = QApplication.primaryScreen().geometry()
        self.setGeometry(150, 150, int(screen.width()*0.2), int(screen.height()*0.2))
        self.setStyleSheet("""
            background-color: #EF5350; 
            border: 5px solid #EF5350;
            border-radius: 10px;
        """)

        layout = QVBoxLayout()
        buttons = [
            ("加载数据", self.load_file),
            ("存档管理", self.open_archive_window),
            ("数据分析", self.open_details_window)
        ]
        
        for text, callback in buttons:
            btn = QPushButton(text)
            btn.setFixedHeight(50)
            btn.setStyleSheet("""
                QPushButton {
                    color: white; font-size: 16px;
                    font-weight: bold;
                    background-color: #E53935;
                    border: 2px solid white;
                    border-radius: 8px;
                    margin: 1px;
                }
                QPushButton:hover { background-color: #D32F2F; }
                QPushButton:pressed { background-color: #B71C1C; }
            """)
            btn.clicked.connect(callback)
            layout.addWidget(btn)
        
        self.setLayout(layout)

    def load_archive_files(self):
        """加载存档文件"""
        archive_dir = "archives"
        if not os.path.exists(archive_dir):
            os.makedirs(archive_dir)
        
        self.archive_files = {}
        for file in os.listdir(archive_dir):
            if file.endswith('.txt'):
                try:
                    with open(os.path.join(archive_dir, file), 'r', encoding='utf-8') as f:
                        self.archive_files[file[:-4]] = f.read().splitlines()
                except Exception as e:
                    logging.error(f"加载存档失败 {file}: {str(e)}")

    def load_file(self):
        """加载数据文件"""
        options = QFileDialog.Option.ReadOnly
        file_name, _ = QFileDialog.getOpenFileName(
            self, "选择数据文件", "", 
            "文本文件 (*.txt *.csv);;所有文件 (*.*)", 
            options=options
        )
        
        if file_name:
            try:
                with open(file_name, 'r', encoding='utf-8') as f:
                    self.data = []
                    for line in f:
                        line = line.split('#')[0].strip()
                        if not line: continue
                        
                        date_period, numbers = self.parser.parse_line(line)
                        formatted = self.parser.format_output(date_period, numbers)
                        if formatted:
                            self.data.append(formatted)
                    
                    if self.data:
                        QMessageBox.information(self, "成功", f"已加载 {len(self.data)} 条有效数据")
                    else:
                        QMessageBox.warning(self, "警告", "未找到符合格式的数据")
            except Exception as e:
                QMessageBox.critical(self, "错误", f"文件读取失败: {str(e)}")

    def open_archive_window(self):
        """打开存档管理窗口"""
        self.archive_dialog = QDialog(self)
        self.archive_dialog.setWindowTitle("存档管理")
        screen = QApplication.primaryScreen().geometry()
        self.archive_dialog.setFixedSize(int(screen.width()*0.25), int(screen.height()*0.25))
        
        layout = QVBoxLayout()
        self.archive_list = QListWidget()
        self.archive_list.addItems(self.archive_files.keys())
        layout.addWidget(self.archive_list)
        
        btn_layout = QHBoxLayout()
        buttons = [
            ("添加存档", "#4CAF50", self.add_archive),
            ("加载存档", "#2196F3", self.load_from_archive),
            ("删除存档", "#F44336", self.delete_archive)
        ]
        
        for text, color, callback in buttons:
            btn = QPushButton(text)
            btn.setFixedHeight(40)
            btn.setStyleSheet(f"""
                QPushButton {{
                    background-color: {color}; color: white;
                    font-weight: bold; border-radius: 5px;
                }}
                QPushButton:hover {{ background-color: {color[:-2] + 'C'}; }}
            """)
            btn.clicked.connect(callback)
            btn_layout.addWidget(btn)
        
        layout.addLayout(btn_layout)
        self.archive_dialog.setLayout(layout)
        self.archive_dialog.exec()

    def add_archive(self):
        """添加新存档"""
        if not self.data:
            QMessageBox.warning(self.archive_dialog, "警告", "没有可存档的数据")
            return
            
        name, ok = QInputDialog.getText(self.archive_dialog, "存档命名", "输入存档名称:")
        if ok and name:
            archive_dir = "archives"
            if not os.path.exists(archive_dir):
                os.makedirs(archive_dir)
                
            try:
                with open(os.path.join(archive_dir, f"{name}.txt"), 'w', encoding='utf-8') as f:
                    f.write("\n".join(self.data))
                
                self.archive_files[name] = self.data.copy()
                self.archive_list.addItem(name)
                QMessageBox.information(self.archive_dialog, "成功", "存档已保存")
            except Exception as e:
                QMessageBox.critical(self.archive_dialog, "错误", f"存档保存失败: {str(e)}")

    def load_from_archive(self):
        """从存档加载数据"""
        selected = self.archive_list.currentItem()
        if selected:
            name = selected.text()
            self.data = self.archive_files[name]
            QMessageBox.information(self.archive_dialog, "成功", f"已加载存档: {name}")
            self.archive_dialog.close()

    def delete_archive(self):
        """删除存档"""
        selected = self.archive_list.currentItem()
        if selected:
            name = selected.text()
            reply = QMessageBox.question(
                self.archive_dialog, "确认删除", 
                f"确定删除存档 {name} 吗？",
                QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No
            )
            if reply == QMessageBox.StandardButton.Yes:
                try:
                    os.remove(os.path.join("archives", f"{name}.txt"))
                    del self.archive_files[name]
                    self.archive_list.takeItem(self.archive_list.row(selected))
                except Exception as e:
                    QMessageBox.critical(self.archive_dialog, "错误", f"删除失败: {str(e)}")

    def open_details_window(self):
        """打开数据分析窗口"""
        if not self.data:
            QMessageBox.warning(self, "警告", "请先加载数据")
            return
            
        try:
            screen = QApplication.primaryScreen().geometry()
            self.data_window = DataWindow(self)
            self.data_window.setGeometry(
                50, 50, int(screen.width() * 0.5), int(screen.height() * 0.5))
            self.data_window.show()
            
            self.console_window = ConsoleWindow(self)
            self.console_window.show()
        except Exception as e:
            QMessageBox.critical(self, "错误", f"打开详情窗口失败: {str(e)}")

class DataWindow(QWidget):
    def __init__(self, lottery_window):
        super().__init__()
        try:
            self.lottery_window = lottery_window
            self.init_ui()
            self.original_display.append(f"\n已加载 {len(self.lottery_window.data)} 期开奖号码")
        except Exception as e:
            QMessageBox.critical(self, "错误", f"数据窗口初始化失败: {str(e)}")

    def init_ui(self):
        self.setWindowTitle("数据详情")
        self.setStyleSheet("background-color: white;")
        
        layout = QHBoxLayout()
        self.original_display = QTextEdit()
        self.original_display.setReadOnly(True)
        self.original_display.setStyleSheet("""
            QTextEdit {
                background-color: #F5F5F5; color: black;
                font-family: 'Courier New'; font-size: 24px;
                font-weight: bold;
                border: 1px solid #BDBDBD; border-radius: 5px;
            }
        """)
        self.original_display.setPlainText("\n".join(self.lottery_window.data))
        
        self.generate_display = QTextEdit()
        self.generate_display.setReadOnly(True)
        self.generate_display.setStyleSheet("""
            QTextEdit {
                background-color: #FFF8E1; color: black;
                font-family: 'Courier New'; font-size: 24px;
                font-weight: bold;
                border: 1px solid #FFD54F; border-radius: 5px;
            }
        """)
        
        self.query_display = QTextEdit()
        self.query_display.setReadOnly(True)
        self.query_display.setStyleSheet("""
            QTextEdit {
                background-color: #E8F5E9; color: black;
                font-family: 'Courier New'; font-size: 24px;
                font-weight: bold;
                border: 1px solid #81C784; border-radius: 5px;
            }
        """)
        
        for widget in [self.original_display, self.generate_display, self.query_display]:
            scroll = QScrollArea()
            scroll.setWidgetResizable(True)
            scroll.setWidget(widget)
            layout.addWidget(scroll)
        
        layout.setStretch(0, 4)
        layout.setStretch(1, 3)
        layout.setStretch(2, 3)
        self.setLayout(layout)

    def wheelEvent(self, event: QWheelEvent):
        if QApplication.keyboardModifiers() == Qt.KeyboardModifier.ControlModifier:
            widget = self.childAt(event.position().toPoint())
            if isinstance(widget, QTextEdit):
                font = widget.font()
                if event.angleDelta().y() > 0:
                    font.setPointSize(font.pointSize() + 1)
                else:
                    font.setPointSize(max(8, font.pointSize() - 1))
                widget.setFont(font)
        else:
            super().wheelEvent(event)

    def update_query_result(self, results):
        """更新查询结果"""
        self.query_display.clear()
        if results:
            self.query_display.setPlainText("\n".join(results))
            self.query_display.append(f"\n共找到 {len(results)} 条匹配记录")

    def update_generate_result(self, results):
        """更新生成结果"""
        self.generate_display.clear()
        if results:
            display_results = results[:1000]
            self.generate_display.setPlainText("\n".join(display_results))
            self.generate_display.append(f"\n共生成 {len(results)} 组号码")
            if len(results) > 1000:
                self.generate_display.append("(仅显示前1000组)")

class ConsoleWindow(QWidget):
    def __init__(self, lottery_window):
        super().__init__()
        try:
            self.lottery_window = lottery_window
            self.filter_status = {
                "4重": False, "3重": False, "2重": False,
                "4连": False, "3连": False, "2连": False,
                "数字大": False, "数字小": False, "数字单": False, "数字双": False,
                "龙": False, "虎": False, "和": False,
                "和值大": False, "和值小": False, "和值单": False, "和值双": False,
                "取/除1": "取", "取/除2": "取"
            }
            self.displayed_data = set()
            self.init_ui()
            self.center_on_screen()
        except Exception as e:
            QMessageBox.critical(self, "错误", f"控制台初始化失败: {str(e)}")

    def center_on_screen(self):
        screen = QApplication.primaryScreen().geometry()
        self.move(
            int((screen.width() - self.width()) / 2),
            int((screen.height() - self.height()) / 2)
        )

    def init_ui(self):
        self.setWindowTitle("彩票控制台")
        self.setStyleSheet("""
            background-color: #121212;
            padding: 0px;
        """)
        screen = QApplication.primaryScreen().geometry()
        self.setFixedSize(int(screen.width()*0.4), int(screen.height()*0.6))
        
        layout = QVBoxLayout()
        layout.setContentsMargins(0, 0, 0, 0)
        layout.setSpacing(0)
        
        self.display = QTextEdit()
        self.display.setReadOnly(True)
        self.display.setStyleSheet("""
            QTextEdit {
                background-color: #1E1E1E; color: #FFD700;
                font-family: 'Courier New'; font-size: 12px;
                border: 1px solid #424242; border-radius: 5px;
                margin: 0px;
            }
        """)
        self.display.setPlainText("本地数据分析控制台")
        
        scroll = QScrollArea()
        scroll.setWidgetResizable(True)
        scroll.setWidget(self.display)
        layout.addWidget(scroll, stretch=4)
        
        control_panel = self.create_control_panel()
        layout.addWidget(control_panel, stretch=3)
        
        self.setLayout(layout)

    def create_control_panel(self):
        panel = QWidget()
        panel.setStyleSheet("""
            background-color: #1E1E1E;
            border-radius: 5px;
            padding: 0px;
        """)
        main_layout = QVBoxLayout(panel)
        main_layout.setContentsMargins(0, 0, 0, 0)
        main_layout.setSpacing(0)

        # 数字输入区域
        digit_group = QWidget()
        digit_layout = QGridLayout(digit_group)
        digit_layout.setHorizontalSpacing(0)
        digit_layout.setVerticalSpacing(0)
        
        positions = ["千位", "百位", "十位", "个位"]
        self.digit_inputs = []
        for i, placeholder in enumerate(positions):
            input_box = QLineEdit()
            input_box.setPlaceholderText(placeholder)
            input_box.setFixedSize(90, 30)
            input_box.setStyleSheet("""
                QLineEdit {
                    color: #FFD700; font-size: 16px;
                    border: 2px solid #424242; border-radius: 5px;
                    background-color: #2D2D2D;
                    margin: 1px;
                }
            """)
            input_box.setAlignment(Qt.AlignmentFlag.AlignCenter)
            input_box.textChanged.connect(self.validate_input)
            input_box.returnPressed.connect(self.generate_numbers)
            self.digit_inputs.append(input_box)
            digit_layout.addWidget(input_box, 0, i)
        
        main_layout.addWidget(digit_group)

        # 筛选按钮网格布局
        filter_grid = QGridLayout()
        filter_grid.setHorizontalSpacing(0)
        filter_grid.setVerticalSpacing(0)

        # 按钮样式
        btn_style = """
            QPushButton {
                background-color: #2D2D2D; color: #FFD700;
                font-weight: bold; font-size: 14px;
                border: 1px solid #424242; border-radius: 3px;
                margin: 1px;
            }
            QPushButton:checked {
                background-color: #424242; color: #FFD700;
            }
            QPushButton:hover {
                background-color: #3D3D3D;
            }
        """
        
        # 特殊按钮样式
        dragon_tiger_style = """
            QPushButton {
                background-color: #9C27B0; color: #FFD700;
                font-weight: bold; font-size: 14px;
                border: 1px solid #7B1FA2; border-radius: 3px;
                margin: 1px;
            }
            QPushButton:checked {
                background-color: #7B1FA2;
            }
            QPushButton:hover {
                background-color: #8E24AA;
            }
        """

        # 4重3重2重除取 - 4行1列
        for i, text in enumerate(["4重", "3重", "2重"]):
            btn = QPushButton(text)
            btn.setCheckable(True)
            btn.setFixedSize(70, 30)
            btn.setStyleSheet(btn_style)
            btn.clicked.connect(lambda _, t=text: self.toggle_filter(t))
            filter_grid.addWidget(btn, i, 0)
        
        # 添加取/除1按钮
        self.add_switch_button(filter_grid, "取/除1", 3, 0)

        # 4连3连2连除取 - 4行1列
        for i, text in enumerate(["4连", "3连", "2连"]):
            btn = QPushButton(text)
            btn.setCheckable(True)
            btn.setFixedSize(70, 30)
            btn.setStyleSheet(btn_style)
            btn.clicked.connect(lambda _, t=text: self.toggle_filter(t))
            filter_grid.addWidget(btn, i, 1)
        
        # 添加取/除2按钮
        self.add_switch_button(filter_grid, "取/除2", 3, 1)

        # 数字大小单双 - 4行1列 (新增的筛选条件)
        for i, text in enumerate(["数字大", "数字小", "数字单", "数字双"]):
            btn = QPushButton(text)
            btn.setCheckable(True)
            btn.setFixedSize(70, 30)
            btn.setStyleSheet(btn_style)
            btn.clicked.connect(lambda _, t=text: self.toggle_filter(t))
            filter_grid.addWidget(btn, i, 2)

        # 龙虎和和值大小单双 - 4行2列
        for i, text in enumerate(["龙", "虎", "和"]):
            btn = QPushButton(text)
            btn.setCheckable(True)
            btn.setFixedSize(70, 30)
            btn.setStyleSheet(dragon_tiger_style)
            btn.clicked.connect(lambda _, t=text: self.toggle_filter(t))
            filter_grid.addWidget(btn, i, 3)

        # 和值大小单双按钮
        for i, text in enumerate(["和值大", "和值小", "和值单", "和值双"]):
            btn = QPushButton(text)
            btn.setCheckable(True)
            btn.setFixedSize(70, 30)
            btn.setStyleSheet(dragon_tiger_style)
            btn.clicked.connect(lambda _, t=text: self.toggle_filter(t))
            filter_grid.addWidget(btn, i, 4)

        # 和值输入框
        self.sum_input = QLineEdit()
        self.sum_input.setPlaceholderText("和值")
        self.sum_input.setFixedSize(70, 30)
        self.sum_input.setValidator(QIntValidator(0, 36))
        self.sum_input.setStyleSheet("""
            QLineEdit {
                color: #FFD700; font-size: 16px;
                border: 1px solid #424242; border-radius: 3px;
                background-color: #2D2D2D;
                margin: 1px;
            }
        """)
        self.sum_input.setAlignment(Qt.AlignmentFlag.AlignCenter)
        filter_grid.addWidget(self.sum_input, 3, 4)

        # 期号筛选输入框和功能按钮区域
        bottom_layout = QHBoxLayout()
        
        # 期号查找筛选输入框
        period_labels = ["起始日期", "结束日期", "起始期号", "结束期号"]
        self.period_inputs = []
        for i, placeholder in enumerate(period_labels):
            input_box = QLineEdit()
            input_box.setPlaceholderText(placeholder)
            if i < 2:
                input_box.setFixedSize(90, 30)
            else:
                input_box.setFixedSize(45, 30)
            input_box.setStyleSheet("""
                QLineEdit {
                    color: #FFD700; font-size: 16px;
                    border: 1px solid #424242; border-radius: 5px;
                    background-color: #2D2D2D;
                    margin: 1px;
                }
            """)
            input_box.setAlignment(Qt.AlignmentFlag.AlignCenter)
            self.period_inputs.append(input_box)
            bottom_layout.addWidget(input_box)
        
        bottom_layout.addStretch()
        
        # 查询按钮
        btn_query = QPushButton("查询")
        btn_query.setFixedSize(90, 30)
        btn_query.setStyleSheet("""
            QPushButton {
                background-color: #2D2D2D; color: #FFD700;
                font-weight: bold; border-radius: 5px;
                border: 1px solid #424242;
            }
            QPushButton:hover { background-color: #3D3D3D; }
        """)
        btn_query.clicked.connect(self.execute_query)
        bottom_layout.addWidget(btn_query)
        
        # 生成按钮
        btn_generate = QPushButton("生成")
        btn_generate.setFixedSize(90, 30)
        btn_generate.setStyleSheet("""
            QPushButton {
                background-color: #2D2D2D; color: #FFD700;
                font-weight: bold; border-radius: 5px;
                border: 1px solid #424242;
            }
            QPushButton:hover { background-color: #3D3D3D; }
        """)
        btn_generate.clicked.connect(self.generate_numbers)
        bottom_layout.addWidget(btn_generate)

        # 将筛选按钮和底部布局放入垂直布局
        v_layout = QVBoxLayout()
        v_layout.addLayout(filter_grid)
        v_layout.addLayout(bottom_layout)
        
        main_layout.addLayout(v_layout)
        return panel

    def add_switch_button(self, layout, name, row, col):
        """创建取/除切换按钮"""
        container = QWidget()
        container.setFixedSize(70, 30)
        container.setStyleSheet("background-color: transparent;")
        
        hbox = QHBoxLayout(container)
        hbox.setContentsMargins(0, 0, 0, 0)
        hbox.setSpacing(0)
        
        # 左边"除"按钮
        btn_remove = QPushButton("除")
        btn_remove.setCheckable(True)
        btn_remove.setFixedSize(35, 30)
        btn_remove.setStyleSheet("""
            QPushButton {
                background-color: #2D2D2D; color: #FFD700;
                font-weight: bold; border: none;
                border-top-left-radius: 3px;
                border-bottom-left-radius: 3px;
            }
            QPushButton:checked {
                background-color: #424242; color: #FFD700;
            }
        """)
        btn_remove.clicked.connect(lambda: self.set_switch_state(name, "除"))
        
        # 右边"取"按钮
        btn_take = QPushButton("取")
        btn_take.setCheckable(True)
        btn_take.setFixedSize(35, 30)
        btn_take.setStyleSheet("""
            QPushButton {
                background-color: #2D2D2D; color: #FFD700;
                font-weight: bold; border: none;
                border-top-right-radius: 3px;
                border-bottom-right-radius: 3px;
            }
            QPushButton:checked {
                background-color: #424242; color: #FFD700;
            }
        """)
        btn_take.clicked.connect(lambda: self.set_switch_state(name, "取"))
        
        hbox.addWidget(btn_remove)
        hbox.addWidget(btn_take)
        layout.addWidget(container, row, col)
        
        # 初始化状态
        self.filter_status[name] = "取"
        btn_take.setChecked(True)

    def set_switch_state(self, name, state):
        """设置取/除切换按钮状态"""
        self.filter_status[name] = state
        container = self.sender().parent()
        for btn in container.findChildren(QPushButton):
            btn.setChecked(btn.text() == state)

    def validate_input(self):
        """输入校验"""
        sender = self.sender()
        text = sender.text()
        unique_digits = []
        for char in text:
            if char.isdigit() and char not in unique_digits:
                unique_digits.append(char)
        sender.setText("".join(unique_digits))

    def toggle_filter(self, filter_name):
        """切换筛选状态（带互斥处理）"""
        # 互斥组处理
        mutex_groups = [
            ["数字大", "数字小"], 
            ["数字单", "数字双"],
            ["和值大", "和值小"],
            ["和值单", "和值双"],
            ["龙", "虎", "和"]
        ]
        
        for group in mutex_groups:
            if filter_name in group:
                for btn_name in group:
                    if btn_name != filter_name:
                        self.filter_status[btn_name] = False
                        # 更新按钮UI状态
                        for child in self.findChildren(QPushButton):
                            if child.text() == btn_name:
                                child.setChecked(False)
        
        self.filter_status[filter_name] = not self.filter_status[filter_name]

    def execute_query(self):
        """执行查询"""
        try:
            digits = [input.text() for input in self.digit_inputs]
            if not any(digits):
                QMessageBox.warning(self, "警告", "请输入查询条件")
                return
                
            active_filters = [name for name, active in self.filter_status.items() 
                            if active and name not in ["取/除1", "取/除2"]]
            matched = []
            
            # 处理期号筛选条件
            period_filters = [input.text() for input in self.period_inputs]
            
            for item in self.lottery_window.data:
                parts = [p.strip() for p in item.split("  ")]
                if len(parts) >= 2:
                    num = parts[1]
                    date_period = parts[0]
                    
                    # 检查期号筛选条件
                    if not self.check_period_filters(date_period, period_filters):
                        continue
                    
                    if self.match_filters(num, digits, active_filters):
                        matched.append(item)
            
            self.lottery_window.data_window.update_query_result(matched)
            self.display.append(f"\n查询完成，共找到 {len(matched)} 条匹配记录")
        except Exception as e:
            self.display.append(f"\n[错误] 查询失败: {str(e)}")

    def check_period_filters(self, date_period, filters):
        """检查期号筛选条件"""
        if not any(filters):
            return True
            
        try:
            date_part = date_period[:8] if len(date_period) >= 8 else ""
            period_part = date_period[8:] if len(date_period) > 8 else ""
            
            # 检查日期范围
            start_date = filters[0]
            end_date = filters[1]
            
            if start_date and date_part < start_date:
                return False
            if end_date and date_part > end_date:
                return False
                
            # 检查期号范围
            start_period = filters[2]
            end_period = filters[3]
            
            if start_period and period_part < start_period:
                return False
            if end_period and period_part > end_period:
                return False
                
            return True
        except:
            return True

    def generate_numbers(self):
        """生成号码"""
        try:
            if not any(input.text() for input in self.digit_inputs):
                QMessageBox.warning(self, "警告", "请输入生成条件")
                return
                
            digit_ranges = []
            for i in range(4):
                input_text = self.digit_inputs[i].text()
                if not input_text:
                    digit_ranges.append([])
                    continue
                    
                unique_digits = []
                for char in input_text:
                    if char.isdigit() and char not in unique_digits:
                        unique_digits.append(char)
                digit_ranges.append(unique_digits)
            
            active_filters = [name for name, active in self.filter_status.items() 
                            if active and name not in ["取/除1", "取/除2"]]
            matched = []
            valid_positions = [i for i in range(4) if digit_ranges[i]]
            
            if valid_positions:
                for combo in product(*[digit_ranges[i] for i in valid_positions]):
                    num = ["0"]*4
                    for idx, pos in enumerate(valid_positions):
                        num[pos] = combo[idx]
                    num_str = "".join(num)
                    
                    # 检查和值条件
                    sum_value = self.sum_input.text()
                    if sum_value:
                        total = sum(int(d) for d in num_str)
                        if total != int(sum_value):
                            continue
                    
                    if self.match_filters(num_str, ["","","",""], active_filters):
                        total = sum(int(d) for d in num_str)
                        matched.append(f"{num_str} 和值:{total}")
            
            self.lottery_window.data_window.update_generate_result(matched)
            self.display.append(f"\n生成完成，共产生 {len(matched)} 组号码")
            if len(matched) > 1000:
                self.display.append("(仅显示前1000组)")
        except Exception as e:
            self.display.append(f"\n[错误] 生成失败: {str(e)}")

    def match_filters(self, num, digits, filters):
        """匹配筛选条件"""
        # 基础数字匹配
        for i in range(4):
            if digits[i] and num[i] not in digits[i]:
                return False
        
        # 龙虎和判断
        first, last = int(num[0]), int(num[3])
        if "龙" in filters and not first > last:
            return False
        if "虎" in filters and not first < last:
            return False
        if "和" in filters and not first == last:
            return False
        
        # 数字大小单双属性 (新增的筛选条件)
        if "数字大" in filters and not all(int(d) >= 5 for d in num):
            return False
        if "数字小" in filters and not all(int(d) < 5 for d in num):
            return False
        if "数字单" in filters and not all(int(d) % 2 == 1 for d in num):
            return False
        if "数字双" in filters and not all(int(d) % 2 == 0 for d in num):
            return False
        
        # 和值属性
        total = sum(int(d) for d in num)
        if "和值大" in filters and total < 18:
            return False
        if "和值小" in filters and total >= 18:
            return False
        if "和值单" in filters and total % 2 == 0:
            return False
        if "和值双" in filters and total % 2 == 1:
            return False
        
        # 重号检测（带取/除逻辑）
        repeat_checks = {
            "4重": len(set(num)) == 1,
            "3重": any(num.count(d) >= 3 for d in num),
            "2重": len(set(num)) < 4
        }
        for name, condition in repeat_checks.items():
            if name in filters:
                if self.filter_status["取/除1"] == "除" and condition:
                    return False
                if self.filter_status["取/除1"] == "取" and not condition:
                    return False
        
        # 连号检测（带取/除逻辑）
        consecutive_checks = {
            "4连": self.has_consecutive(num, 4),
            "3连": self.has_consecutive(num, 3),
            "2连": self.has_consecutive(num, 2)
        }
        for name, condition in consecutive_checks.items():
            if name in filters:
                if self.filter_status["取/除2"] == "除" and condition:
                    return False
                if self.filter_status["取/除2"] == "取" and not condition:
                    return False
        
        return True

    def has_consecutive(self, num, length):
        """检查连续数字（支持循环连续如7890）"""
        # 标准连续检测
        for i in range(len(num)-length+1):
            if all(int(num[i+j]) == int(num[i])+j for j in range(length)):
                return True
        
        # 特殊循环连续检测
        if length >= 3:
            cyclic_segments = {
                3: ["890", "901", "987", "098"],
                4: ["7890", "8901", "9876", "0987"]
            }
            if "".join(num) in cyclic_segments.get(length, []):
                return True
        
        return False

if __name__ == "__main__":
    try:
        app = QApplication(sys.argv)
        font = QFont("Microsoft YaHei")
        font.setPointSize(10)
        font.setBold(True)
        app.setFont(font)
        main_window = MainApp()
        main_window.show()
        sys.exit(app.exec())
    except Exception as e:
        logging.exception("程序崩溃")
        QMessageBox.critical(None, "致命错误", f"程序崩溃: {str(e)}")
