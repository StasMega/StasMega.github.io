<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rich Editor</title>
    <link rel="stylesheet" href="https://unpkg.com/easymde/dist/easymde.min.css">
    <script src="https://unpkg.com/easymde/dist/easymde.min.js"></script>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <style>
        body { font-family: sans-serif; margin: 0; padding: 10px; background-color: var(--tg-theme-bg-color); color: var(--tg-theme-text-color); }
        .editor-preview { background: #fff !important; color: #000 !important; }
        #send-btn {
            display: block; width: 100%; padding: 15px; margin-top: 10px;
            background-color: var(--tg-theme-button-color);
            color: var(--tg-theme-button-text-color);
            border: none; border-radius: 8px; font-weight: bold; cursor: pointer;
        }
    </style>
</head>
<body>

    <textarea id="markdown-editor"></textarea>
    <button id="send-btn">ОТПРАВИТЬ КАК RICH MESSAGE</button>

    <script>
        const tg = window.Telegram.WebApp;
        tg.expand(); 
        const easyMDE = new EasyMDE({
            element: document.getElementById('markdown-editor'),
            placeholder: "Напиши что-нибудь классное с #заголовками и |таблицами|...",
            spellChecker: false,
            autosave: { enabled: true, uniqueId: "rich-editor-v1" },
            toolbar: ["bold", "italic", "heading", "|", "quote", "unordered-list", "table", "|", "preview", "side-by-side"]
        });

        document.getElementById('send-btn').addEventListener('click', () => {
            const content = easyMDE.value();
            if (!content.trim()) {
                alert("Сначала напиши что-нибудь!");
                return;
            }
            tg.sendData(content);
        });
    </script>
</body>
</html>
