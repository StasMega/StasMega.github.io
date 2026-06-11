<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Rich Editor Pro</title>
    
    <!-- Стили для редактора -->
    <link rel="stylesheet" href="https://unpkg.com/easymde/dist/easymde.min.css">
    <script src="https://unpkg.com/easymde/dist/easymde.min.js"></script>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    
    <style>
        :root {
            --bg-color: var(--tg-theme-bg-color, #ffffff);
            --text-color: var(--tg-theme-text-color, #000000);
            --hint-color: var(--tg-theme-hint-color, #999999);
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            margin: 0;
            padding: 0;
            display: flex;
            flex-direction: column;
            height: 100vh;
            overflow: hidden;
        }

        /* Настройка внешнего вида редактора под тему Telegram */
        .editor-toolbar {
            background: var(--tg-theme-secondary-bg-color, #f0f0f0);
            border: none;
            border-bottom: 1px solid var(--tg-theme-hint-color, #ccc);
            opacity: 1;
        }
        .editor-toolbar button {
            color: var(--text-color) !important;
        }
        .editor-toolbar button.active, .editor-toolbar button:hover {
            background: var(--tg-theme-bg-color);
            border: none;
        }

        .CodeMirror {
            border: none;
            background: var(--bg-color);
            color: var(--text-color);
            font-size: 16px;
            flex-grow: 1;
        }

        /* Кастомный превью, похожий на облачко Telegram */
        .editor-preview {
            background: var(--tg-theme-secondary-bg-color, #e7ebf0) !important;
            padding: 15px !important;
        }
        
        .preview-bubble {
            background: var(--tg-theme-bg-color, #fff);
            border-radius: 12px;
            padding: 10px 15px;
            box-shadow: 0 1px 2px rgba(0,0,0,0.1);
            color: #000; /* Внутри чата текст обычно черный/белый */
            max-width: 90%;
            margin: 0 auto;
        }

        #editor-container {
            flex-grow: 1;
            display: flex;
            flex-direction: column;
        }
    </style>
</head>
<body>

    <div id="editor-container">
        <textarea id="md-editor"></textarea>
    </div>

    <script>
        const tg = window.Telegram.WebApp;
        tg.ready();
        tg.expand();

        // Инициализация EasyMDE
        const easyMDE = new EasyMDE({
            element: document.getElementById('md-editor'),
            spellChecker: false,
            autosave: { enabled: true, uniqueId: "rich-editor-v2" },
            toolbar: ["bold", "italic", "strikethrough", "|", "heading", "table", "quote", "|", "side-by-side", "preview"],
            status: false,
            minHeight: "100%",
            placeholder: "Введите текст поста...",
        });

        // Настройка кнопки MainButton (синяя кнопка внизу Telegram)
        const mainButton = tg.MainButton;
        mainButton.setText("ОТПРАВИТЬ В ЧАТ");
        mainButton.show();

        // Следим за изменениями текста, чтобы показывать/скрывать кнопку
        easyMDE.codemirror.on("change", () => {
            if (easyMDE.value().trim().length > 0) {
                mainButton.show();
            } else {
                mainButton.hide();
            }
        });

        // Обработка нажатия на главную кнопку
        mainButton.onClick(() => {
            const content = easyMDE.value();
            
            // Вибрация (Haptic Feedback) — телефон слегка "дзынькнет"
            tg.HapticFeedback.notificationOccurred('success');
            
            // Показываем лоадер на кнопке
            mainButton.showProgress();
            
            // Отправляем данные боту
            tg.sendData(content);
        });

        // Адаптация под темную тему
        function applyTheme() {
            const isDark = tg.colorScheme === 'dark';
            // Можно добавить специфичные стили для темной темы через JS
            document.body.style.setProperty('--bg-color', tg.themeParams.bg_color);
        }

        tg.onEvent('themeChanged', applyTheme);
        applyTheme();
    </script>
</body>
</html>
