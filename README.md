# tire-check-tool

**Disclaimer:** Это учебный проект, созданный в рамках курса по ML-ops, цель которого - освоить базовые инструменты для разработки и деплоя ML-моделей. *Модель, используемая в проекте, является "игрушечной" и не предназначена для использования в реальных условиях.*

## Description

Учебный проект по курсу ML-ops. В рамках проекта реализован python-пакет для проверки шин автомобиля на износ по фотографии. Пакет содержит в себе модель для классификации изображений, а также CLI-интерфейс для работы с ней:

- обучения модели (train)
- предсказания класса изображения (infer)
- экспорта модели в ONNX-формат (export)
- запуска оценки выбранного изображения на сервере MLflow (run_server)


Пример использования пакета:
![image](img/tire-check-tool_demo.png)

## Installation

Для использования пакета необходимо установить его из репозитория:

```bash
git clone https://github.com/VitalyyBezuglyj/tire-check-tool.git

cd tire-check-tool

# Вероятно, вы захотите сделать это в виртуальном окружении
# Например, используя conda:
# conda create -n tire-check-tool python=3.8
pip install -e .
```

## Usage

*Note:* Такие команды как `train` и `run_server` - предполагают, что MLflow запущен и доступен по адресу localhost:5000. Подробнее см раздел [MLflow](#mlflow).

После установки пакета в командной строке доступны следующие команды:

- `python commands.py train` - обучение модели
    - Скачает датасет, если он еще не скачан (~ $5$ Гб)

- `python commands.py infer` - оценка точности модели на всем датасете
    - по-умолчанию используются веса модели, полученные при обучении
    - если указать аргумент `--use_pretrained` загрузит предобученные веса и использует их

- `python commands.py export` - экспорт модели в ONNX-формат
    - точно также - по-умолчанию - веса от вашего обучения, с флагом `--use_pretrained` - предобученные веса (скачиваются автоматически)

- `python commands.py run_server --image <path/to/image>` - запуск оценки выбранного изображения на сервере MLflow
    - `--image` - путь к изображению, по-умолчанию - `img/demo_tire_image.jpg`
    - сначала необходимо выполнить экспорт модели в ONNX-формат, см. команду `export`


### Конфигурация

Конфигурация модели и обучения задается в файле [config.yaml](configs/default.yaml).

Любая команда поддерживает следующие аргументы:

- `--config_name` - имя файла конфигурации в директории [configs](configs), по-умолчанию - `default`

- `--config_path` - путь к директории с конфигурационными файлами, по-умолчанию - `configs`

- Любой аргумент из конфигурационного файла, например `--epochs 10` или `--batch_size 32`, если аргумент вложенный, то используется точка в качестве разделителя, например `--tire_check_model.learning_rate 0.001`

- `-- --help` - справка по команде, дополнительный аргумент `--` необходим для разделения аргументов команды и аргументов CLI

### MLflow

1. Если у вас откуда-то есть запущенный сервер MLflow, то вам необходимо указать его адрес в файле [config.yaml](configs/default.yaml) в параметре `loggers.mlflow.tracking_uri` или указать при запуске в виде аргумента командной строки `--loggers.mlflow.tracking_uri <ваш адрес>`. По-умолчанию используется `http://localhost:5000`.

2. Если у вас нет запущенного сервера MLflow, то его можно запустить локально:

```bash
# Install MLflow
pip install mlflow

# Run MLflow
mlflow server --host localhost --port 5000 --artifacts-destination outputs/mlflow_artifacts
```

Соответственно, artifacts-destination - это директория, в которой будут храниться различные логи и пр, и её можно выбрать произвольно.

## Dataset

Для обучения модели использовался датасет [Tire-Check](https://www.kaggle.com/datasets/warcoder/tyre-quality-classification), содержащий 2 класса изображений шин автомобиля: `good` ($831$) и `defective` ($1 031$). Фото сделаны на довольно близком расстоянии, и признаком износа, по видимому, является наличие трещин на шине.

Примеры изображений:

![image](img/tires_dataset_demo.png)

Для обучения данные были разделены на тренировочную и валидационную выборки в соотношении 80/20. Для тестирования модели (infer) использовалась вся выборка.

*Данные для обучения будут скачаны автоматически при запуске обучения/валидации и размещены в директории data*

## Model

Задача игрушечная, а в программу курса входило использование torch lightning, поэтому модель была написана с нуля и единственная её цель - показать работу пайплайна и более-менее сходящиеся метрики. Что она и делает.

Архитектура модели:

```python
TireCheckModel(
  (conv1): Conv2d(3, 16, kernel_size=(3, 3), stride=(1, 1))
  (pool): MaxPool2d(kernel_size=2, stride=2, padding=0, dilation=1, ceil_mode=False)
  (conv2): Conv2d(16, 32, kernel_size=(3, 3), stride=(1, 1))
  (pool): MaxPool2d(kernel_size=2, stride=2, padding=0, dilation=1, ceil_mode=False)
  (conv3): Conv2d(32, 64, kernel_size=(3, 3), stride=(1, 1))
  (pool): MaxPool2d(kernel_size=2, stride=2, padding=0, dilation=1, ceil_mode=False)
  (conv4): Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1))
  (pool): MaxPool2d(kernel_size=3, stride=3, padding=0, dilation=1, ceil_mode=False)
  (fc1): Linear(in_features=2304, out_features=512, bias=True)
  (fc2): Linear(in_features=512, out_features=2, bias=True)
)
```

Лучший результат на валидации:
{'epoch': 16, 'MulticlassF1Score': 0.7826}

Эти веса могут быть использованы для экспорта/инференса модели при указании аргумента `--use_pretrained` в командной строке или конфиге.
