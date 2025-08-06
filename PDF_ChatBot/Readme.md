
https://ollama.com/library/deepseek-r1:1.5b

i have ollama on mac and downloaded deepseek-r1:1.5b model. help with a agent that call the model with specific tool (access to a pdf document that is local machine) and create a chatbot on the document. The agent should have a good UI with choice to use different models on the laptop (default is deepseek-r1:1.5b). the UI should ask for if need to (1) generate a summary, (2) generate a MCQ . the generated agent code should be modular, clean , follow SOLID design principles, have exception handling and logging. code should be idiomatic

Here’s a complete, modular, and idiomatic Python application (using PyQt5 for the UI) for macOS that interacts with Ollama’s local system models (defaulting to deepseek-r1:1.5b) and provides an agent for PDF-based Q&A, summarization, and MCQ generation. The application architecture emphasizes SOLID principles, clean code, logging, and exception handling.

**Key Features:**
- **PDF upload** and parsing.
- **Choice of local LLM model** (from available Ollama models).
- **Chat, summary, or MCQ generation** options, with a friendly GUI.
- **Modular design** with clear separation of UI, logic, and integration code.

## 1. Installation Requirements

```bash
pip install PyQt5 PyPDF2 requests
```

## 2. Modular Project Structure

```
ollama_pdf_agent/
├── main.py
├── ui.py
├── agent.py
├── pdf_parser.py
├── ollama_client.py
├── logger.py
└── config.py
```

### 3. logger.py

```python
import logging

def get_logger(name="ollama_pdf_agent"):
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)
    if not logger.handlers:
        handler = logging.StreamHandler()
        formatter = logging.Formatter('[%(asctime)s] %(levelname)s: %(message)s')
        handler.setFormatter(formatter)
        logger.addHandler(handler)
    return logger
```

### 4. config.py

```python
OLLAMA_ENDPOINT = "http://localhost:11434/api/generate"
AVAILABLE_MODELS = ["deepseek-r1:1.5b", "llama2", "mistral", "other-loaded-models"]
```

### 5. pdf_parser.py

```python
import PyPDF2
from logger import get_logger

logger = get_logger()

class PDFParser:
    @staticmethod
    def extract_text(file_path: str) -> str:
        try:
            with open(file_path, 'rb') as file:
                reader = PyPDF2.PdfReader(file)
                text = "".join(page.extract_text() or "" for page in reader.pages)
            if not text:
                logger.warning("PDF parsed but returned empty.")
            return text
        except Exception as e:
            logger.exception("PDF parsing failed.")
            raise RuntimeError("Could not parse PDF.") from e
```

### 6. ollama_client.py

```python
import requests
from config import OLLAMA_ENDPOINT
from logger import get_logger

logger = get_logger()

class OllamaClient:
    def __init__(self, endpoint=OLLAMA_ENDPOINT):
        self.endpoint = endpoint

    def generate(self, prompt: str, model: str):
        try:
            response = requests.post(self.endpoint, json={"model": model, "prompt": prompt})
            response.raise_for_status()
            content = response.json().get("response", "")
            logger.info(f"Model response: {content[:80]}...")
            return content
        except Exception as e:
            logger.exception("Ollama model call failed.")
            raise RuntimeError("Ollama generation failed.") from e
```

### 7. agent.py

```python
from ollama_client import OllamaClient
from pdf_parser import PDFParser
from logger import get_logger

logger = get_logger()

class PDFChatAgent:
    def __init__(self, pdf_text: str, model: str = "deepseek-r1:1.5b"):
        self.pdf_text = pdf_text
        self.client = OllamaClient()
        self.model = model

    def chat(self, query: str) -> str:
        prompt = f"Document:\n{self.pdf_text[:4000]}\n---\nQuestion: {query}\nAnswer:"
        return self.client.generate(prompt, self.model)

    def summarize(self) -> str:
        prompt = f"Summarize the following document:\n{self.pdf_text[:4000]}"
        return self.client.generate(prompt, self.model)

    def generate_mcq(self, n_questions: int = 5) -> str:
        prompt = f"Create {n_questions} multiple-choice questions with 4 options and answers from the following text:\n{self.pdf_text[:4000]}"
        return self.client.generate(prompt, self.model)
```

### 8. ui.py

```python
from PyQt5.QtWidgets import (QWidget, QFileDialog, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QTextEdit, QComboBox, QApplication, QRadioButton, QButtonGroup)
from PyQt5.QtCore import Qt
from config import AVAILABLE_MODELS
from pdf_parser import PDFParser
from agent import PDFChatAgent
from logger import get_logger

logger = get_logger()

class MainWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Ollama PDF Chatbot")
        self.setGeometry(100, 100, 700, 600)

        self.pdf_text = ""
        self.agent = None

        layout = QVBoxLayout()

        self.model_combo = QComboBox()
        self.model_combo.addItems(AVAILABLE_MODELS)
        layout.addWidget(QLabel("Select Model:"))
        layout.addWidget(self.model_combo)

        self.load_pdf_btn = QPushButton("Load PDF")
        self.load_pdf_btn.clicked.connect(self.load_pdf)
        layout.addWidget(self.load_pdf_btn)

        self.option_group = QButtonGroup()
        self.radio_chat = QRadioButton("Chat")
        self.radio_summary = QRadioButton("Summary")
        self.radio_mcq = QRadioButton("MCQ")
        self.radio_chat.setChecked(True)
        for btn in [self.radio_chat, self.radio_summary, self.radio_mcq]:
            self.option_group.addButton(btn)
        radio_layout = QHBoxLayout()
        radio_layout.addWidget(self.radio_chat)
        radio_layout.addWidget(self.radio_summary)
        radio_layout.addWidget(self.radio_mcq)
        layout.addLayout(radio_layout)

        self.question_input = QTextEdit()
        self.question_input.setPlaceholderText("Type your question here (for chat mode)...")
        layout.addWidget(self.question_input)

        self.run_btn = QPushButton("Run")
        self.run_btn.clicked.connect(self.run_agent)
        layout.addWidget(self.run_btn)

        self.result_display = QTextEdit()
        self.result_display.setReadOnly(True)
        layout.addWidget(self.result_display)

        self.setLayout(layout)

    def load_pdf(self):
        file_path, _ = QFileDialog.getOpenFileName(self, "Open PDF", "", "PDF Files (*.pdf)")
        if file_path:
            try:
                self.pdf_text = PDFParser.extract_text(file_path)
                self.agent = PDFChatAgent(pdf_text=self.pdf_text, model=self.model_combo.currentText())
                self.result_display.setPlainText("PDF loaded successfully!")
                logger.info("PDF loaded.")
            except Exception as e:
                self.result_display.setPlainText("Failed to load PDF.")
                logger.error(f"Failed to load PDF: {e}")

    def run_agent(self):
        if not self.agent:
            self.result_display.setPlainText("Please load a PDF first.")
            return
        option = None
        if self.radio_chat.isChecked():
            option = "chat"
        elif self.radio_summary.isChecked():
            option = "summary"
        elif self.radio_mcq.isChecked():
            option = "mcq"

        try:
            if option == "chat":
                query = self.question_input.toPlainText().strip()
                if not query:
                    self.result_display.setPlainText("Enter a question.")
                    return
                response = self.agent.chat(query)
            elif option == "summary":
                response = self.agent.summarize()
            elif option == "mcq":
                response = self.agent.generate_mcq()
            self.result_display.setPlainText(response)
        except Exception as e:
            self.result_display.setPlainText(str(e))
            logger.error(f"Agent error: {e}")
```

### 9. main.py

```python
import sys
from PyQt5.QtWidgets import QApplication
from ui import MainWindow

def main():
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec_())

if __name__ == '__main__':
    main()
```

## Usage

1. Start Ollama server and ensure your desired models are available locally.
2. Place your files as per the project structure above.
3. Run:

```bash
python main.py
```

**This will bring up the UI. Upload your PDF, select the model, and pick the mode (chat/summary/MCQ). Interact freely—the design is modular, robust, extensible, and idiomatic Python.**
