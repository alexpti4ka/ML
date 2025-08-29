# Layout Parser

## Описание проекта

Система автоматического парсинга документов с распознаванием структуры и извлечением данных в формат JSON.

## Задача

Обработка входных документов (фото/PDF) с различными типами контента:
- Таблицы с данными
- Заголовки и подзаголовки  
- Текстовые блоки
- Логотипы и изображения

**Результат**: структурированный JSON с распознанными элементами и их содержимым.

## Пример задачи
На вход — инженерная таблица (table_example.jpeg)
На выход — структурированный JSON, содержащий не только значения таблицы, но и важные нетабличные элементы, если такие есть (шапки, заголовки, текстовые блоки, логотипы).

Язык — русский (!)


## Архитектура

### 1. Layout-анализ (выделение блоков документа)

- **Вход:**  
  jpg/png/pdf файл (скан или фото документа)

- **Технологии:**  
  Table Transformer (detector) / Donut / Amazon Textract Layout API / Azure Form Recognizer Layout

- **Что делает:**  
  Находит и классифицирует на изображении все главные блоки: таблицы, текстовые зоны, подписи, печати, изображения; возвращает координаты этих блоков.

- **Выход (пример):**
```
[
  {"type": "table", "bbox": [50, 120, 550, 820]},
  {"type": "text", "bbox": [30, 30, 770, 110]},
  {"type": "text", "bbox": [570, 870, 900, 1030]},
  {"type": "picture", "bbox": [800, 100, 950, 250]}
]
```

### 2. Table Extraction (распознавание структуры таблицы)

- **Вход:**  
Изображение выделенной таблицы (crop по bbox из layout-модуля для типа "table")

- **Технологии:**  
Table Transformer / DeepDeSRT / CascadeTabNet / Amazon Textract Tables API

- **Что делает:**  
Разбивает табличную область на строки/столбцы/ячейки (в том числе объединённые), строит структуру, распознаёт текст по каждой ячейке.

- **Выход (пример):**
```
[
["Обозначение", "Название", "Q, м3/ч"],
["ВТЗ-1", "Помещение 116", "1700"],
...
]
```

### 3. OCR нетабличных областей (шапка, подписи, прочее)

- **Вход:**  
Изображения нетабличных блоков (crop из общего файла для областей =! "table")

- **Технологии:**  
docTR / Tesseract / EasyOCR / Google Vision OCR

- **Что делает:**  
Распознаёт и возвращает текст из всех не-табличных зон документа, в т.ч. с распознанных как картинка (логотипы, схемы, графики).

- **Выход (пример):**
```
{
"header": "Характеристика систем",
"object": "Магазин, Новосибирск"
}
```

### 4. Интеграция и маппинг (постобработка)

- **Вход:**  
Табличные данные (из модуля 2), текстовые блоки (из модуля 3)

- **Технологии:**  
pandas / custom Python-скрипты / словари маппинга / simple NLP

- **Что делает:**  
Объединяет все распознанные данные в единую структуру (json/csv/xlsx), приводит названия полей к стандарту, готовит к экспорту/анализу.

Находит заголовки, подзаголовки и обычный текст по высоте, площади, расположению на странице. Также используются правила длины (заголовки короче), регулярные выражения, анализ отступов.

- **Выход (пример):**
```
{
"metadata": { ... },
"table": [
{"system": "ВТЗ-1", "room": "Помещение 116", "air_flow": 1700, ...},
...
]
}
```

## Технический стек

### Core ML
- **Layout Detection**: YOLO/Detectron2 для сегментации блоков
- **OCR**: Tesseract/EasyOCR для текстового содержимого  
- **Table Parsing**: Table Transformer для структуры таблиц
- **Logo Detection**: Custom CNN для распознавания брендов

### Backend
- **Python 3.9+** 
- **FastAPI** - REST API
- **OpenCV** - обработка изображений
- **PIL/Pillow** - работа с форматами
- **pdf2image** - конвертация PDF


## Этапы разработки

### Phase 1: MVP
- [ ] Базовая детекция блоков (текст/таблица/изображение)
- [ ] OCR с Tesseract
- [ ] Простой JSON выход
- [ ] FastAPI setup

### Phase 2: Enhanced Detection  
- [ ] Улучшенная сегментация с Detectron2
- [ ] Специализированный парсинг таблиц
- [ ] Детекция логотипов
- [ ] Классификация типов текста

### Phase 3: Production
- [ ] Оптимизация производительности
- [ ] Batch processing
- [ ] Error handling и валидация
- [ ] Monitoring и логирование


## Настройка среды

```
conda create -n ml_env python=3.11
conda activate ml_env
conda install numpy pandas scipy scikit-learn matplotlib jupyterlab
conda install pytorch torchvision torchaudio -c pytorch
pip install transformers
pip install python-dotenv tqdm optuna lightgbm catboost xgboost
pip install pytesseract opencv-python matplotlib
brew install tesseract tesseract-lang
```

## Конфигурация (.env) (раздел в разработке)

```env
# API Settings
API_HOST=0.0.0.0
API_PORT=8000
DEBUG=false

# ML Models
LAYOUT_MODEL_PATH=./models/layout_detectron2.pth
OCR_LANGUAGE=rus+eng
TABLE_MODEL_PATH=./models/table_transformer.pth

# Processing
MAX_FILE_SIZE=50MB
PROCESSING_TIMEOUT=300
TEMP_DIR=/tmp/layout_parser

# Storage
OUTPUT_DIR=./data/output
LOG_LEVEL=INFO
```

## Структура проекта (раздел в разработке)

```
layout-parser/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI приложение
│   ├── models/
│   │   ├── layout_model.py  # Модель детекции блоков
│   │   ├── ocr_model.py     # OCR обработка
│   │   └── table_model.py   # Парсинг таблиц
│   ├── services/
│   │   ├── document_processor.py
│   │   ├── layout_detector.py
│   │   └── json_formatter.py
│   └── schemas/
│       └── document_schema.py
├── models/                  # Веса ML моделей
├── tests/
├── data/
│   ├── input/              # Тестовые документы
│   └── output/             # JSON результаты
├── requirements.txt
├── .env.example
└── README.md
```


## Безопасность (раздел в разработке)

⚠️ **Важные моменты:**
- Ограничение размера загружаемых файлов
- Валидация форматов документов
- Очистка временных файлов после обработки
- Rate limiting для API
- Логирование всех операций


