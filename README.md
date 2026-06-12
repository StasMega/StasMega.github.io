<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Native Rich Editor Pro</title>
    <!-- Telegram SDK -->
    <script src="https://telegram.org/js/telegram-web-app.js"></script>
    <!-- Иконки -->
    <script src="https://unpkg.com/@phosphor-icons/web"></script>
    
    <style>
        /* Нативные цвета Telegram */
        :root {
            --bg: var(--tg-theme-bg-color, #ffffff);
            --secondary-bg: var(--tg-theme-secondary-bg-color, #f0f0f0);
            --text: var(--tg-theme-text-color, #000000);
            --hint: var(--tg-theme-hint-color, #999999);
            --btn: var(--tg-theme-button-color, #3390ec);
            --btn-text: var(--tg-theme-button-text-color, #ffffff);
            --border: rgba(128, 128, 128, 0.2);
            --list-bg: var(--tg-theme-bg-color, #ffffff);
        }

        body, html {
            margin: 0; padding: 0; height: 100%;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--secondary-bg); color: var(--text);
            display: flex; flex-direction: column;
            -webkit-tap-highlight-color: transparent;
            overflow: hidden;
        }

        /* Верхняя панель (только для простого текста) */
        .toolbar {
            display: flex; overflow-x: auto; padding: 10px 15px;
            background: var(--bg); border-bottom: 1px solid var(--border);
            gap: 10px; scrollbar-width: none; flex-shrink: 0;
            box-shadow: 0 1px 3px rgba(0,0,0,0.05);
        }
        .toolbar::-webkit-scrollbar { display: none; }
        
        .tool-btn {
            background: var(--secondary-bg); border: none; padding: 8px 12px;
            color: var(--text); font-size: 20px; border-radius: 8px;
            cursor: pointer; display: flex; align-items: center; justify-content: center;
            transition: 0.15s; flex-shrink: 0;
        }
        .tool-btn:active { transform: scale(0.92); opacity: 0.8; }
        .tool-btn * { pointer-events: none; }

        .divider { width: 1px; background: var(--hint); opacity: 0.3; margin: 0 5px; flex-shrink: 0; }

        /* Поле редактора */
        .editor-container {
            flex: 1; overflow-y: auto; padding: 20px; padding-bottom: 100px;
            font-size: 17px; line-height: 1.5; outline: none;
            background: var(--bg); color: var(--text);
        }
        .editor-container[placeholder]:empty:before { content: attr(placeholder); color: var(--hint); pointer-events: none; }

        /* Плавающая кнопка "Добавить" (+) */
        .fab {
            position: fixed; bottom: 20px; right: 20px;
            width: 56px; height: 56px; border-radius: 28px;
            background: var(--btn); color: var(--btn-text);
            display: flex; align-items: center; justify-content: center;
            font-size: 28px; box-shadow: 0 4px 12px rgba(0,0,0,0.3);
            cursor: pointer; z-index: 100; transition: 0.2s; border: none;
        }
        .fab:active { transform: scale(0.9); }

        /* Всплывающее меню (Bottom Sheet) */
        .bottom-sheet {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.5); z-index: 1000;
            display: flex; flex-direction: column; justify-content: flex-end;
            opacity: 0; pointer-events: none; transition: 0.3s;
        }
        .bottom-sheet.show { opacity: 1; pointer-events: auto; }
        
        .sheet-content {
            background: var(--secondary-bg); width: 100%;
            border-radius: 20px 20px 0 0; padding-bottom: 30px;
            transform: translateY(100%); transition: 0.3s cubic-bezier(0.1, 0.8, 0.2, 1);
        }
        .bottom-sheet.show .sheet-content { transform: translateY(0); }

        .sheet-header {
            display: flex; justify-content: space-between; align-items: center;
            padding: 15px 20px; font-weight: 600; font-size: 18px; border-bottom: 1px solid var(--border);
        }
        .close-btn { background: transparent; border: none; color: var(--hint); font-size: 24px; cursor: pointer; }

        /* Списки в стиле BotFather */
        .tg-list { background: var(--bg); margin-top: 10px; }
        .tg-item {
            display: flex; align-items: center; padding: 12px 20px;
            border-bottom: 1px solid var(--border); cursor: pointer; transition: 0.1s;
        }
        .tg-item:active { background: var(--secondary-bg); }
        .tg-item:last-child { border-bottom: none; }
        
        .tg-icon-wrap {
            width: 40px; height: 40px; border-radius: 50%;
            display: flex; align-items: center; justify-content: center;
            color: white; font-size: 22px; margin-right: 15px; flex-shrink: 0;
        }
        .tg-text-block { flex: 1; display: flex; flex-direction: column; }
        .tg-title { font-weight: 500; font-size: 16px; margin-bottom: 2px; }
        .tg-sub { font-size: 13px; color: var(--hint); }
        .tg-arrow { color: var(--hint); font-size: 20px; }

        /* Сетка для таблицы (теперь большая и удобная) */
        .grid-container {
            display: grid; grid-template-columns: repeat(8, 1fr); gap: 5px;
            padding: 20px; touch-action: none; background: var(--bg);
        }
        .grid-cell {
            aspect-ratio: 1; border: 1.5px solid var(--hint);
            border-radius: 6px; transition: 0.1s;
        }
        .grid-cell.active { background: var(--btn); border-color: var(--btn); opacity: 0.9; }
        .grid-status { text-align: center; padding: 10px; font-weight: bold; font-size: 16px; background: var(--bg); }

        /* Оформление элементов внутри редактора */
        table { width: 100%; border-collapse: collapse; margin: 15px 0; background: var(--secondary-bg); color: var(--text); border-radius: 8px; overflow: hidden; }
        td, th { border: 1px solid var(--border); padding: 12px; text-align: left; }
        th { background: rgba(128, 128, 128, 0.1); font-weight: bold; }
        pre { background: var(--secondary-bg); padding: 12px; border-radius: 8px; font-family: monospace; overflow-x: auto; color: var(--text); border: 1px solid var(--border); }
        
        /* Скрытие экранов в модалке */
        .sheet-view { display: none; }
        .sheet-view.active { display: block; }
    </style>
</head>
<body>

    <!-- Панель простого форматирования -->
    <div class="toolbar" id="toolbar">
        <button class="tool-btn action-btn" data-cmd="bold"><i class="ph ph-text-b"></i></button>
        <button class="tool-btn action-btn" data-cmd="italic"><i class="ph ph-text-italic"></i></button>
        <button class="tool-btn action-btn" data-cmd="underline"><i class="ph ph-text-underline"></i></button>
        <button class="tool-btn action-btn" data-cmd="strikeThrough"><i class="ph ph-text-strikethrough"></i></button>
        <div class="divider"></div>
        <button class="tool-btn action-btn" data-block="H1"><i class="ph ph-text-h-one"></i></button>
        <button class="tool-btn action-btn" data-block="H2"><i class="ph ph-text-h-two"></i></button>
        <div class="divider"></div>
        <button class="tool-btn action-btn" data-cmd="insertUnorderedList"><i class="ph ph-list-bullets"></i></button>
    </div>

    <!-- Редактор -->
    <div class="editor-container" id="editor" contenteditable="true" placeholder="Напишите ваш текст здесь..."></div>

    <!-- Плавающая кнопка добавления элементов -->
    <button class="fab" id="fab-btn"><i class="ph ph-plus"></i></button>

    <!-- Bottom Sheet (Всплывающее меню) -->
    <div class="bottom-sheet" id="bottom-sheet">
        <div class="sheet-content" onclick="event.stopPropagation()">
            
            <!-- Экран 1: Список элементов (Стиль BotFather) -->
            <div class="sheet-view active" id="view-menu">
                <div class="sheet-header">
                    <span>Добавить элемент</span>
                    <button class="close-btn" onclick="closeSheet()"><i class="ph ph-x"></i></button>
                </div>
                <div class="tg-list">
                    <!-- Таблица -->
                    <div class="tg-item" onclick="openGridSelector()">
                        <div class="tg-icon-wrap" style="background: var(--btn);"><i class="ph ph-table"></i></div>
                        <div class="tg-text-block">
                            <div class="tg-title">Таблица</div>
                            <div class="tg-sub">Выбрать размер и вставить</div>
                        </div>
                        <i class="ph ph-caret-right tg-arrow"></i>
                    </div>
                    <!-- Код -->
                    <div class="tg-item" onclick="insertCodeBlock()">
                        <div class="tg-icon-wrap" style="background: #34C759;"><i class="ph ph-code"></i></div>
                        <div class="tg-text-block">
                            <div class="tg-title">Блок кода</div>
                            <div class="tg-sub">Для форматирования скриптов</div>
                        </div>
                    </div>
                </div>
            </div>

            <!-- Экран 2: Выбор размера таблицы -->
            <div class="sheet-view" id="view-grid">
                <div class="sheet-header">
                    <span id="grid-status">Выберите размер</span>
                    <button class="close-btn" onclick="closeSheet()"><i class="ph ph-x"></i></button>
                </div>
                <div class="grid-container" id="grid-container"></div>
            </div>

        </div>
    </div>

    <script>
        const tg = window.Telegram.WebApp;
        tg.ready();
        tg.expand();

        const editor = document.getElementById('editor');
        const mainButton = tg.MainButton;
        mainButton.setText("ОТПРАВИТЬ ПОСТ");

        // --- Сохранение позиции курсора (Важно для мобилок!) ---
        let savedRange = null;
        function saveSelection() {
            if (window.getSelection && window.getSelection().rangeCount > 0) {
                savedRange = window.getSelection().getRangeAt(0);
            }
        }
        function restoreSelection() {
            if (savedRange && window.getSelection) {
                let sel = window.getSelection();
                sel.removeAllRanges();
                sel.addRange(savedRange);
            }
        }

        // --- Обработка верхних кнопок ---
        document.querySelectorAll('.action-btn').forEach(btn => {
            btn.addEventListener('mousedown', e => e.preventDefault()); // Не теряем фокус
            btn.addEventListener('click', function() {
                tg.HapticFeedback.selectionChanged();
                if (this.dataset.cmd) document.execCommand(this.dataset.cmd, false, null);
                else if (this.dataset.block) document.execCommand('formatBlock', false, this.dataset.block);
                checkContent();
            });
        });

        // --- Логика Bottom Sheet ---
        const sheet = document.getElementById('bottom-sheet');
        const viewMenu = document.getElementById('view-menu');
        const viewGrid = document.getElementById('view-grid');

        document.getElementById('fab-btn').addEventListener('click', () => {
            tg.HapticFeedback.impactOccurred('light');
            saveSelection(); // Запоминаем, где стоял курсор
            viewMenu.classList.add('active');
            viewGrid.classList.remove('active');
            sheet.classList.add('show');
        });

        function closeSheet() {
            sheet.classList.remove('show');
            setTimeout(() => editor.focus(), 300); // Возвращаем фокус после закрытия
        }
        sheet.addEventListener('click', closeSheet); // Закрытие по клику на фон

        // --- Логика вставки КОДА ---
        function insertCodeBlock() {
            restoreSelection();
            document.execCommand('insertHTML', false, '<br><pre><code>// Ваш код здесь...</code></pre><p><br></p>');
            closeSheet();
            checkContent();
        }

        // --- Логика сетки ТАБЛИЦЫ ---
        function openGridSelector() {
            viewMenu.classList.remove('active');
            viewGrid.classList.add('active');
            updateGrid(0, 0); // Сбрасываем сетку
        }

        const MAX_ROWS = 8, MAX_COLS = 8;
        const gridCont = document.getElementById('grid-container');
        const statusText = document.getElementById('grid-status');
        let selRow = 0, selCol = 0;

        // Рисуем сетку 8х8
        for (let r = 1; r <= MAX_ROWS; r++) {
            for (let c = 1; c <= MAX_COLS; c++) {
                let cell = document.createElement('div');
                cell.className = 'grid-cell';
                cell.dataset.r = r; cell.dataset.c = c;
                gridCont.appendChild(cell);
            }
        }

        function updateGrid(r, c) {
            selRow = r; selCol = c;
            statusText.textContent = (r > 0 && c > 0) ? `Таблица ${c} x ${r}` : 'Выберите размер';
            Array.from(gridCont.children).forEach(cell => {
                if (cell.dataset.r <= r && cell.dataset.c <= c && r > 0 && c > 0) {
                    cell.classList.add('active');
                } else {
                    cell.classList.remove('active');
                }
            });
        }

        // Свайп по сетке (Мобилки)
        gridCont.addEventListener('touchmove', (e) => {
            e.preventDefault(); 
            let touch = e.touches[0];
            let element = document.elementFromPoint(touch.clientX, touch.clientY);
            if (element && element.classList.contains('grid-cell')) {
                tg.HapticFeedback.selectionChanged();
                updateGrid(parseInt(element.dataset.r), parseInt(element.dataset.c));
            }
        });

        // Движение мышкой (ПК)
        gridCont.addEventListener('mouseover', (e) => {
            if (e.target.classList.contains('grid-cell')) {
                updateGrid(parseInt(e.target.dataset.r), parseInt(e.target.dataset.c));
            }
        });

        // Подтверждение выбора таблицы
        function finalizeGrid(e) {
            e.preventDefault();
            if (selRow > 0 && selCol > 0) {
                restoreSelection(); // Возвращаем курсор на место
                tg.HapticFeedback.notificationOccurred('success');
                
                let html = '<br><table style="width:100%; border-collapse: collapse; margin: 15px 0;"><tbody>';
                for (let r = 0; r < selRow; r++) {
                    html += '<tr>';
                    for (let c = 0; c < selCol; c++) {
                        if (r === 0) html += `<th>Заголовок</th>`;
                        else html += `<td>Текст</td>`;
                    }
                    html += '</tr>';
                }
                html += '</tbody></table><p><br></p>';
                
                document.execCommand('insertHTML', false, html);
                closeSheet();
                checkContent();
            }
        }

        gridCont.addEventListener('click', finalizeGrid);
        gridCont.addEventListener('touchend', finalizeGrid);

        // --- Главная кнопка отправки ---
        function checkContent() {
            if (editor.textContent.trim().length > 0 || editor.innerHTML.includes('<table')) {
                if (!mainButton.isVisible) mainButton.show();
            } else {
                if (mainButton.isVisible) mainButton.hide();
            }
        }
        editor.addEventListener('input', checkContent);

        mainButton.onClick(() => {
            tg.HapticFeedback.notificationOccurred('success');
            tg.sendData(editor.innerHTML);
        });
    </script>
</body>
</html>
