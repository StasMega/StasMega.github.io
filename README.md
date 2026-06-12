<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Native Rich Editor</title>
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <script src="https://unpkg.com/@phosphor-icons/web"></script>
    
    <style>
        :root {
            --bg: var(--tg-theme-bg-color, #ffffff);
            --text: var(--tg-theme-text-color, #000000);
            --hint: var(--tg-theme-hint-color, #999999);
            --btn: var(--tg-theme-button-color, #3390ec);
            --border: var(--tg-theme-secondary-bg-color, #efeff3);
        }

        body, html {
            margin: 0; padding: 0; height: 100%;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg); color: var(--text);
            display: flex; flex-direction: column;
            -webkit-tap-highlight-color: transparent;
        }

        .toolbar {
            display: flex; overflow-x: auto; padding: 12px 15px;
            background: var(--bg); border-bottom: 1px solid var(--border);
            gap: 12px; scrollbar-width: none;
            position: sticky; top: 0; z-index: 10;
        }
        .toolbar::-webkit-scrollbar { display: none; }
        
        .tool-btn {
            background: var(--border); border: none; padding: 8px 12px;
            color: var(--text); font-size: 22px; border-radius: 10px;
            cursor: pointer; display: flex; align-items: center; justify-content: center;
            transition: 0.1s; flex-shrink: 0;
        }
        .tool-btn:active { transform: scale(0.95); background: var(--hint); }

        .divider { width: 1px; background: var(--hint); opacity: 0.3; margin: 0 5px; flex-shrink: 0; }

        .editor-container {
            flex: 1; overflow-y: auto; padding: 20px;
            font-size: 17px; line-height: 1.5; outline: none;
        }
        .editor-container[placeholder]:empty:before {
            content: attr(placeholder); color: var(--hint); pointer-events: none;
        }

        table { width: 100%; border-collapse: collapse; margin: 15px 0; background: var(--border); border-radius: 8px; overflow: hidden; }
        td, th { border: 1px solid var(--hint); padding: 10px; text-align: left; }
        th { background: rgba(0,0,0,0.05); font-weight: bold; }
        blockquote { border-left: 4px solid var(--btn); margin: 10px 0; padding-left: 15px; color: var(--hint); font-style: italic; }
        pre { background: var(--border); padding: 12px; border-radius: 8px; font-family: monospace; overflow-x: auto; }
        

        .map-placeholder {
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            background: var(--border); height: 120px; border-radius: 12px; margin: 15px 0;
            color: var(--hint); border: 2px dashed var(--hint); pointer-events: none;
        }
        .map-placeholder i { font-size: 32px; margin-bottom: 5px; }
    </style>
</head>
<body>

    <div class="toolbar" id="toolbar">
        <button class="tool-btn" onclick="format('bold')"><i class="ph ph-text-b"></i></button>
        <button class="tool-btn" onclick="format('italic')"><i class="ph ph-text-italic"></i></button>
        <button class="tool-btn" onclick="format('underline')"><i class="ph ph-text-underline"></i></button>
        <button class="tool-btn" onclick="format('strikeThrough')"><i class="ph ph-text-strikethrough"></i></button>
        
        <div class="divider"></div>
        
        <button class="tool-btn" onclick="formatBlock('H1')"><i class="ph ph-text-h-one"></i></button>
        <button class="tool-btn" onclick="formatBlock('H2')"><i class="ph ph-text-h-two"></i></button>
        <button class="tool-btn" onclick="format('insertUnorderedList')"><i class="ph ph-list-bullets"></i></button>
        
        <div class="divider"></div>
        
        <button class="tool-btn" onclick="insertTable()"><i class="ph ph-table"></i> Таблица</button>
        <button class="tool-btn" onclick="formatBlock('PRE')"><i class="ph ph-code"></i> Код</button>
        <button class="tool-btn" onclick="insertMap()"><i class="ph ph-map-pin"></i> Карта</button>
    </div>

    <div class="editor-container" id="editor" contenteditable="true" placeholder="Начни печатать тут или добавь таблицу сверху..."></div>

    <script>
        const tg = window.Telegram.WebApp;
        tg.ready();
        tg.expand();

        const editor = document.getElementById('editor');
        const mainButton = tg.MainButton;
        mainButton.setText("ОТПРАВИТЬ ПОСТ");

        editor.addEventListener('input', () => {
            if (editor.textContent.trim().length > 0 || editor.innerHTML.includes('<table') || editor.innerHTML.includes('map-placeholder')) {
                if (!mainButton.isVisible) mainButton.show();
            } else {
                if (mainButton.isVisible) mainButton.hide();
            }
        });

        function format(command) {
            document.execCommand(command, false, null);
            editor.focus();
        }


        function formatBlock(tag) {
            document.execCommand('formatBlock', false, tag);
            editor.focus();
        }

        function insertTable() {
            const tableHTML = `
                <br>
                <table>
                    <tr><th>Стиль</th><th>Результат</th></tr>
                    <tr><td>Неон</td><td>Светящийся</td></tr>
                </table>
                <p><br></p>
            `;
            document.execCommand('insertHTML', false, tableHTML);
            editor.focus();
        }

        function insertMap() {
            const mapHTML = `
                <br>
                <div class="map-placeholder" contenteditable="false" data-is-map="true">
                    <i class="ph ph-map-trifold"></i>
                    <span>Карта (Tallinn)</span>
                </div>
                <p><br></p>
            `;
            document.execCommand('insertHTML', false, mapHTML);
            editor.focus();
        }

        mainButton.onClick(() => {
            tg.HapticFeedback.notificationOccurred('success');
            
            let finalHTML = editor.innerHTML;
            
            finalHTML = finalHTML.replace(
                /<div class="map-placeholder".*?<\/div>/g, 
                '<tg-map lat="59.4370" long="24.7536" zoom="14"></tg-map>'
            );

            try {
                tg.sendData(finalHTML);
            } catch(e) {
                tg.showAlert("Ошибка: Откройте бота через клавиатуру снизу, а не через меню.");
            }
        });
    </script>
</body>
</html>
