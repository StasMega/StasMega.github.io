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
            display: flex; overflow-x: auto; padding: 10px 12px;
            background: var(--bg); border-bottom: 1px solid var(--border);
            gap: 8px; scrollbar-width: none;
            position: sticky; top: 0; z-index: 10; align-items: center;
        }
        .toolbar::-webkit-scrollbar { display: none; }
        
        .tool-btn {
            background: transparent; border: none; padding: 8px 10px;
            color: var(--text); font-size: 22px; border-radius: 8px;
            cursor: pointer; display: flex; align-items: center; justify-content: center;
            transition: 0.1s; flex-shrink: 0; font-family: inherit;
        }
        .tool-btn:active { background: var(--border); }
        .tool-btn.with-text { font-size: 16px; font-weight: 500; gap: 6px; background: var(--border); }
        
        /* ВАЖНО: Делаем так, чтобы клик приходился на саму кнопку, а не на иконку внутри нее */
        .tool-btn * { pointer-events: none; }

        .divider { width: 1px; height: 24px; background: var(--hint); opacity: 0.3; margin: 0 4px; flex-shrink: 0; }

        .tool-dropdown { position: relative; display: flex; }
        .table-popover {
            position: absolute; top: calc(100% + 5px); left: 0;
            background: var(--bg); border: 1px solid var(--border);
            border-radius: 12px; box-shadow: 0 8px 24px rgba(0,0,0,0.15);
            padding: 12px; display: none; flex-direction: column; align-items: center;
            z-index: 100;
        }
        .table-popover.show { display: flex; }
        
        .grid-container { display: grid; grid-template-columns: repeat(8, 22px); gap: 3px; touch-action: none; }
        .grid-cell { width: 22px; height: 22px; border: 1px solid var(--hint); border-radius: 3px; transition: 0.05s; }
        .grid-cell.active { background: var(--btn); border-color: var(--btn); opacity: 0.8; }
        .grid-status { margin-top: 10px; font-size: 14px; font-weight: bold; color: var(--text); }

        .editor-container {
            flex: 1; overflow-y: auto; padding: 20px;
            font-size: 17px; line-height: 1.5; outline: none; color: var(--text);
        }
        .editor-container[placeholder]:empty:before { content: attr(placeholder); color: var(--hint); pointer-events: none; }

        table { width: 100%; border-collapse: collapse; margin: 15px 0; background: var(--bg); color: var(--text); }
        td, th { border: 1px solid var(--hint); padding: 10px; text-align: left; color: var(--text); }
        th { background: var(--border); font-weight: bold; }
        pre { background: var(--border); padding: 12px; border-radius: 8px; font-family: monospace; overflow-x: auto; color: var(--text); }
    </style>
</head>
<body>

    <div class="toolbar" id="toolbar">
        <button class="tool-btn action-btn" data-cmd="bold"><i class="ph ph-text-b"></i></button>
        <button class="tool-btn action-btn" data-cmd="italic"><i class="ph ph-text-italic"></i></button>
        <button class="tool-btn action-btn" data-cmd="underline"><i class="ph ph-text-underline"></i></button>
        <button class="tool-btn action-btn" data-cmd="strikeThrough"><i class="ph ph-text-strikethrough"></i></button>
        
        <div class="divider"></div>
        
        <button class="tool-btn action-btn" data-block="H1"><i class="ph ph-text-h-one"></i></button>
        <button class="tool-btn action-btn" data-block="H2"><i class="ph ph-text-h-two"></i></button>
        <button class="tool-btn action-btn" data-cmd="insertUnorderedList"><i class="ph ph-list-bullets"></i></button>
        
        <div class="divider"></div>
        
        <div class="tool-dropdown">
            <button class="tool-btn with-text" id="btn-table">
                <i class="ph ph-table"></i> Таблица
            </button>
            <div class="table-popover" id="table-popover">
                <div class="grid-container" id="grid-container"></div>
                <div class="grid-status" id="grid-status">Выберите размер</div>
            </div>
        </div>

        <button class="tool-btn with-text action-btn" data-block="PRE"><i class="ph ph-code"></i> Код</button>
    </div>

    <div class="editor-container" id="editor" contenteditable="true" placeholder="Начни печатать тут..."></div>

    <script>
        // ЕСЛИ ЧТО-ТО СЛОМАЕТСЯ, БРАУЗЕР ВЫДАСТ НАМ ОШИБКУ НА ЭКРАН!
        window.onerror = function(msg, url, line) {
            alert("Ошибка скрипта: " + msg + " (строка " + line + ")");
            return false;
        };

        const tg = window.Telegram.WebApp;
        tg.ready();
        tg.expand();

        const editor = document.getElementById('editor');
        const mainButton = tg.MainButton;
        mainButton.setText("ОТПРАВИТЬ ПОСТ");

        // 1. ВАЖНО: Запрещаем кнопкам отбирать фокус у текста!
        document.querySelectorAll('.tool-btn').forEach(btn => {
            btn.addEventListener('mousedown', function(e) {
                e.preventDefault(); // Останавливает потерю фокуса
            });
        });

        // 2. Обработка кнопок форматирования
        document.querySelectorAll('.action-btn').forEach(btn => {
            btn.addEventListener('click', function() {
                if (this.dataset.cmd) {
                    document.execCommand(this.dataset.cmd, false, null);
                } else if (this.dataset.block) {
                    document.execCommand('formatBlock', false, this.dataset.block);
                }
            });
        });

        // 3. Логика таблицы
        const MAX_ROWS = 8, MAX_COLS = 8;
        const gridCont = document.getElementById('grid-container');
        const popover = document.getElementById('table-popover');
        const statusText = document.getElementById('grid-status');
        const btnTable = document.getElementById('btn-table');
        let selRow = 0, selCol = 0;

        for (let r = 1; r <= MAX_ROWS; r++) {
            for (let c = 1; c <= MAX_COLS; c++) {
                let cell = document.createElement('div');
                cell.className = 'grid-cell';
                cell.dataset.r = r;
                cell.dataset.c = c;
                gridCont.appendChild(cell);
            }
        }

        btnTable.addEventListener('click', () => {
            popover.classList.toggle('show');
            if(popover.classList.contains('show')) updateGrid(0, 0);
        });

        document.addEventListener('click', (e) => {
            if (!e.target.closest('.tool-dropdown')) popover.classList.remove('show');
        });

        function updateGrid(r, c) {
            selRow = r; selCol = c;
            statusText.textContent = (r > 0 && c > 0) ? `${c} x ${r}` : 'Выберите размер';
            
            Array.from(gridCont.children).forEach(cell => {
                if (cell.dataset.r <= r && cell.dataset.c <= c && r > 0 && c > 0) {
                    cell.classList.add('active');
                } else {
                    cell.classList.remove('active');
                }
            });
        }

        gridCont.addEventListener('mouseover', (e) => {
            if (e.target.classList.contains('grid-cell')) {
                updateGrid(parseInt(e.target.dataset.r), parseInt(e.target.dataset.c));
            }
        });

        gridCont.addEventListener('touchmove', (e) => {
            e.preventDefault(); 
            let touch = e.touches[0];
            let element = document.elementFromPoint(touch.clientX, touch.clientY);
            if (element && element.classList.contains('grid-cell')) {
                updateGrid(parseInt(element.dataset.r), parseInt(element.dataset.c));
            }
        });

        function finalizeGrid(e) {
            e.preventDefault();
            if (selRow > 0 && selCol > 0) {
                let html = '<br><table style="width:100%; border-collapse: collapse; margin: 15px 0;"><tbody>';
                for (let r = 0; r < selRow; r++) {
                    html += '<tr>';
                    for (let c = 0; c < selCol; c++) {
                        if (r === 0) {
                            html += `<th style="border: 1px solid var(--hint); padding: 10px; background: var(--border);">Заголовок</th>`;
                        } else {
                            html += `<td style="border: 1px solid var(--hint); padding: 10px;">Ячейка</td>`;
                        }
                    }
                    html += '</tr>';
                }
                html += '</tbody></table><p><br></p>';
                
                document.execCommand('insertHTML', false, html);
                popover.classList.remove('show');
                checkContent();
            }
        }

        gridCont.addEventListener('click', finalizeGrid);
        gridCont.addEventListener('touchend', finalizeGrid);

        // 4. Логика кнопки отправки
        function checkContent() {
            if (editor.textContent.trim().length > 0 || editor.innerHTML.includes('<table')) {
                if (!mainButton.isVisible) mainButton.show();
            } else {
                if (mainButton.isVisible) mainButton.hide();
            }
        }

        editor.addEventListener('input', checkContent);

        mainButton.onClick(() => {
            if (tg.HapticFeedback) tg.HapticFeedback.notificationOccurred('success');
            const finalHTML = editor.innerHTML;
            try {
                tg.sendData(finalHTML);
            } catch(e) {
                alert("Ошибка отправки: " + e.message);
            }
        });
    </script>
</body>
</html>
