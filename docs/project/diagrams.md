# Диаграммы

## Диаграмма последовательности

Диаграмма соответствует [UC-01 «Автоматическая цензура изображения»](requirements.md#uc-01): редактор передаёт файл, приложение вызывает ML-модель, скрывает выбранные объекты и предоставляет результат.

```mermaid
sequenceDiagram
    actor Editor as Редактор
    participant App as Приложение ACMS Censor
    participant Model as ML-модель обнаружения объектов

    Editor->>App: Передать изображение, категории и способ скрытия
    App->>App: Проверить файл и параметры
    alt Некорректный файл или параметры
        App-->>Editor: Сообщить причину отказа
    else Входные данные корректны
        App->>Model: Проанализировать изображение
        alt Ошибка модели
            Model-->>App: Сообщить об ошибке анализа
            App-->>Editor: Сообщить, что обработка не завершена
        else Анализ завершён
            Model-->>App: Вернуть категории и координаты объектов
            App->>App: Отобрать объекты выбранных категорий
            alt Найдены совпадения
                App->>App: Скрыть области выбранным способом
            else Совпадений нет
                App->>App: Подготовить копию без изменений
            end
            App->>App: Сохранить отдельный выходной файл
            alt Обработка и сохранение успешны
                App-->>Editor: Предоставить файл и итог обработки
                Editor->>Editor: Сохранить и проверить изображение
            else Ошибка обработки или сохранения
                App-->>Editor: Сообщить об ошибке без готового результата
            end
        end
    end
```

## Диаграмма классов

Модель содержит пять сущностей: исходный медиафайл, параметры цензуры, задание обработки, обнаруженный объект и результат. Для [UC-01](requirements.md#uc-01) рассматриваются изображения и координаты визуальных объектов.

```mermaid
classDiagram
    class MediaFile {
        +string id
        +string name
        +string mediaType
        +string sourcePath
    }
    class CensorSettings {
        +string[] categories
        +string method
    }
    class ProcessingJob {
        +string id
        +string status
        +string errorMessage
        +start()
    }
    class Detection {
        +string category
        +float x
        +float y
        +float width
        +float height
    }
    class ProcessingResult {
        +string outputPath
        +int censoredObjectCount
        +bool unchanged
    }

    ProcessingJob "0..*" --> "1" MediaFile : обрабатывает
    ProcessingJob "1" *-- "1" CensorSettings : содержит
    ProcessingJob "1" *-- "0..*" Detection : получает при анализе
    ProcessingJob "1" *-- "0..1" ProcessingResult : создаёт при успехе
```

Один исходный файл может обрабатываться несколько раз с разными параметрами. Задание содержит один набор параметров и от нуля до нескольких обнаружений. При успешной обработке оно создаёт один результат; при ошибке результат отсутствует. Если объектов выбранных категорий нет, результат содержит путь к копии без изменений, `censoredObjectCount = 0` и `unchanged = true`.
