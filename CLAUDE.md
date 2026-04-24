# MeetScribe — Claude Code Context

## Что это за проект

Браузерное веб-приложение (single-page, без сборщика) для захвата аудио браузерной вкладки (Google Meet, Zoom и т.п.) и транскрибации в реальном времени. Транскрипции сохраняются локально через File System Access API. Никакого облака, никакого сервера.

---

## Структура файлов

```
meetscribe/
├── index.html            # Всё приложение — HTML + CSS + JS в одном файле
├── audio-processor.js    # AudioWorklet processor — ОБЯЗАТЕЛЬНО отдельный файл
├── translations.js       # i18n словари (en/ru) + t(), setLang(), getCurrentLang()
├── prompts/              # Папка с промтами для суммаризации (создаётся пользователем)
│   └── meeting-minutes.txt
├── CLAUDE.md             # Этот файл
├── SPEC.md               # Полная спецификация
└── README.md
```

**Важно:** `audio-processor.js` вынесен отдельно потому что `AudioWorklet.addModule()` требует отдельный файл — Blob URL нестабильно работает в Chrome. Не переносить в `index.html`.

`translations.js` вынесен отдельно для удобства добавления новых языков. Всё остальное — стили, логика, разметка — в `index.html`. Не создавать дополнительных JS/CSS файлов.

---

## Запуск

```bash
npx serve .
# → http://localhost:3000
```

`file://` не работает — File System Access API и AudioWorklet требуют localhost или HTTPS.

---

## Архитектура хранения: папка на сессию

Каждая запись хранится в **отдельной папке** внутри корневой директории:

```
~/Documents/MeetScribe/
├── 2026-04-15_14-31/
│   ├── 2026-04-15_14-31.txt           ← ASR транскрипт
│   ├── 2026-04-15_14-31.webm          ← аудио
│   ├── 2026-04-15_14-31_dg.txt        ← Deepgram AI транскрипция (опционально)
│   ├── 2026-04-15_14-31_ai.txt        ← Gemini AI транскрипция (опционально)
│   ├── 2026-04-15_14-31_chat.json     ← история чата (опционально)
│   └── screenshots/                   ← скриншоты для анализа (опционально)
├── 2026-04-15_14-31_product-vision/   ← после AI-rename (вся папка переименовывается)
│   ├── 2026-04-15_14-31_product-vision.txt
│   ├── 2026-04-15_14-31_product-vision.webm
│   └── 2026-04-15_14-31_product-vision_dg.txt
├── prompts/
│   └── meeting-minutes.txt
└── backgrounds/
```

- `loadSessions()` итерирует поддиректории корневой папки, фильтрует по паттерну `^\d{4}-\d{2}-\d{2}_\d{2}-\d{2}`, ищет `baseTxt = dirName + '.txt'` внутри
- `renameSession(oldFolderName, newSlug)` — копирует все файлы с заменой префикса, удаляет старую папку, возвращает `{ name, dirHandle }`
- `deleteSessionFiles(dirName)` — `S.dirHandle.removeEntry(dirName, { recursive: true })`

---

## Ключевые технические решения

### STT движки

Выбирается в настройках (`meetscribe_live_engine` в localStorage). Каждый движок имеет собственные функции `start*()` и `stop*()`.

**Gemini Live** (`models/gemini-2.5-flash-native-audio-latest`):

```javascript
function buildSetup() {
  return {
    setup: {
      model: 'models/gemini-2.5-flash-native-audio-latest',
      generationConfig: {
        responseModalities: ['AUDIO'],          // AUDIO обязательно для native audio модели
        thinkingConfig: { thinkingBudget: 0 },
      },
      inputAudioTranscription: {},              // на уровне setup, НЕ внутри generationConfig!
      systemInstruction: { parts: [{ text: sysText }] },
    }
  };
}
```

- `inputAudioTranscription: {}` только на уровне `setup` — иначе ошибка 1007
- `responseModalities: ['AUDIO']` — не `['TEXT']`
- Ответы в `serverContent.inputTranscription.text`
- Нельзя слать `{ clientContent: { turnComplete: true } }` — Gemini отклоняет ошибкой 1007
- WebSocket: `wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1alpha.GenerativeService.BidiGenerateContent?key={KEY}`

**Deepgram Live** (`nova-2-general`):
- WebSocket: `wss://api.deepgram.com/v1/listen?encoding=linear16&sample_rate=16000&...`
- `is_final=true` в сообщении → фиксировать реплику
- Параметр `keyterm` для глоссария (не `keywords`)

**Groq** (Whisper Large v3 Turbo):
- Буферизация PCM в `_groqBuffer[]`, flush по тишине или max длине
- PCM → WAV через `pcmToWav()`, POST на `api.groq.com/openai/v1/audio/transcriptions`
- `setLive(t('live_processing'))` во время запроса, `setLive(t('live_listening'))` после

**Web Speech API**:
- `new SpeechRecognition()`, `interimResults: true`, `continuous: true`
- Язык: `navigator.language || 'ru-RU'`
- Только микрофон — вкладка не слушается

### Hard Reset Watchdog (Gemini Live)

Если сервер перестаёт отвечать — watchdog принудительно закрывает сокет, `onclose` запускает реконнект:

```javascript
// В energyInterval (100ms):
if (S.ws && S.wsReady && S.lastActivityTime) {
  const idleTime = Date.now() - S.lastActivityTime;
  if (idleTime > 15000 && instantEnergy > 0.01) {
    S.lastActivityTime = Date.now(); // не срабатывать дважды
    S.ws.close();
  }
}
```

### REST API для суммаризации и AI-rename

Провайдеры: **Gemini** / **Mistral** / **OpenAI** / **Custom** (OpenAI-compatible URL).

Функция `callWithModelOption({ provider, model }, body)`:
- Gemini → прямой `generateContent` запрос
- Остальные → `callOpenAICompat(url, key, prompt, model)`
- Вспомогательная `extractPromptText(body)` — достаёт текст из `body.contents[0].parts[0].text`

Retry при 429/503: задержки `[0, 5000, 12000, 25000]` мс, 4 попытки.

**Текущая рабочая модель Gemini REST: `gemini-2.5-flash`** (подтверждена через ListModels).

Ключи: `localStorage.getItem('meetscribe_rest_key_' + provider)`.

### Суммаризация (✨)

Читает и пишет в `aiFile` (`_dg.txt` или `_ai.txt`) если есть, иначе в базовый `.txt`.

`{{transcript}}` в тексте промта заменяется содержимым файла. Если плейсхолдер отсутствует — транскрипция добавляется в конец через `\n\n` (защита от кастомных промтов без плейсхолдера).

При повторной суммаризации — удаляет старую преамбулу:

```javascript
const meetIdx = baseLines.findIndex(l => l.startsWith('MeetScribe'));
const tsIdx = baseLines.findIndex(l => /^\[\d+:\d+:\d+\]/.test(l));
const cutIdx = meetIdx >= 0 ? meetIdx : (tsIdx >= 0 ? tsIdx : 0);
const transcriptPart = baseLines.slice(cutIdx).join('\n');
```

### Chat с транскриптом (💬)

- История персистируется в `_chat.json` внутри папки сессии
- `S.chat.dirHandle` — handle папки сессии (не корневой!)
- **Context snapshot** при отправке: `ctx = { chatFile, dirHandle, history, model, transcript }` — ответ всегда записывается в правильный файл даже при переключении сессии
- `closeChat()` делает `.blur()` на `#chat-input` — иначе Alt-хоткеи не работают
- Gemini требует строго чередующихся ролей — мержим подряд идущие сообщения одной роли

### AI-транскрипция через внешние сервисы (🎙)

- **Deepgram** (`meetscribe_deepgram_key`) → `_dg.txt`, nova-3, диаризация, параметр `keyterm` (не `keywords`)
- **Gemini AI** (fallback) → `_ai.txt`, Gemini Files API (загрузка → ожидание ACTIVE → `generateContent`)
  - URI кэшируется в localStorage на 47 часов
  - Таймаут ожидания ACTIVE: 80 попыток × 3 секунды = 4 минуты

### Анализ экранов (🖼)

- `analyzeScreenshots(name, dirHandle, modelOpt)` — находит `[SCREENSHOT: fname]` маркеры в `.txt`
- Для каждого скриншота: ±15 секунд транскрипта как контекст → `callVision({ provider, model }, imageB64, prompt)`
- `callVision` поддерживает Gemini (inline_data) и OpenAI-compatible (image_url)
- Mistral → `pixtral-large-latest` модель
- Результаты вставляются обратно в `.txt` файл

### Аудио: PCM 16kHz + WebM/Opus запись

- `AudioContext({ sampleRate: 16000 })` — единый контекст
- `audio-processor.js` конвертирует Float32 → Int16 PCM
- Параллельно: `MediaRecorder` пишет WebM/Opus (`S.recDest` от `createMediaStreamDestination()`)
- Название вкладки из `videoTrack.label` → используется в имени папки сессии

### Определение говорящего

```javascript
// Peak floor (0.015 RMS) предотвращает ложные "You" от фонового шума
if (total > 0 && S.micTurnEnergy / total >= threshold && S.micPeakEnergy >= 0.015) speaker = 'You';
```

`S.micPeakEnergy` — максимальный мгновенный RMS микрофона за реплику. Сбрасывается при каждом `turnComplete`.

### Переподключение вкладки (reattachTabAudio)

```javascript
async function reattachTabAudio() {
  const hint = showCaptureHint();
  const newStream = await navigator.mediaDevices.getDisplayMedia({ audio: true, video: true });
  hint.remove();
  // подключает новый источник к существующему workletNode и tabAnalyser
  // MediaRecorder продолжает запись без перезапуска
}
```

### Файлы: File System Access API

- `DirectoryHandle` сохраняется в IndexedDB (не localStorage — не сериализуется)
- Транскрипт `.txt` создаётся сразу при старте, пишется инкрементально
- Rename = copy + delete (нативного rename нет)

### Бэкап и восстановление

`BACKUP_LS_KEYS` — список всех ключей localStorage для бэкапа:
- Все API ключи, провайдеры, URL
- `meetscribe_live_engine`, `meetscribe_lang`, `meetscribe_chat_model`
- Настройки темы, наушников, панели, фона

`createBackup()` — сохраняет в папку `backup_YYYY-MM-DD/` и скачивает на ПК.
`restoreBackup(file)` — восстанавливает настройки + глоссарий, применяет язык через `setLang()` + `applyTranslations()`.

---

## Состояние глобального объекта S (ключевые поля)

```javascript
S = {
  // directory
  dirHandle,          // FileSystemDirectoryHandle — корневая папка транскрипций
  // API settings
  apiKey,             // ключ для Live API (Gemini)
  restProvider,       // 'gemini' | 'mistral' | 'openai' | 'custom'
  restCustomUrl,      // URL для custom провайдера
  // streams / audio
  displayStream,      // MediaStream от getDisplayMedia
  micStream,          // MediaStream от getUserMedia
  audioCtx,           // AudioContext 16kHz
  workletNode,        // AudioWorkletNode
  sttMode,            // 'gemini' | 'deepgram' | 'groq' | 'webspeech'
  // speaker detection
  tabAnalyser,        // AnalyserNode для вкладки
  micAnalyser,        // AnalyserNode для микрофона
  tabTurnEnergy,      // накопленная энергия вкладки за текущую реплику
  micTurnEnergy,      // накопленная энергия микрофона за текущую реплику
  micPeakEnergy,      // максимальный мгновенный RMS микрофона за реплику
  energyInterval,     // setInterval для анализа энергии (100ms)
  lastActivityTime,   // Date.now() последнего сообщения от сервера (watchdog)
  // audio recording
  mediaRecorder,      // MediaRecorder для .webm
  audioFileName,      // имя .webm файла
  recStartTime,       // Date.now() старта записи
  recDest,            // MediaStreamAudioDestinationNode для записи
  // websocket (Gemini Live / Deepgram)
  ws,                 // WebSocket
  wsReady,            // true после setupComplete (Gemini) / open (Deepgram)
  reconnectAttempts,
  sessionEnded,       // флаг чтобы не реконнектить после явного завершения
  // file
  fileHandle,         // FileSystemFileHandle текущего .txt
  writable,           // FileSystemWritableFileStream для текущего .txt
  fileName,           // имя текущего .txt файла
  recDirHandle,       // FileSystemDirectoryHandle для папки текущей сессии
  recDirName,         // имя папки текущей сессии
  // transcript
  lines,              // массив завершённых реплик
  liveBuf,            // текст текущей незавершённой реплики
  startTs,            // Date.now() старта (для временных меток)
  // chat
  chat: {
    active,           // boolean — чат открыт
    fileName,         // базовое .txt имя сессии
    chatFile,         // имя _chat.json файла
    dirHandle,        // handle папки сессии (не корневой!)
    transcript,       // текст транскрипции для API контекста
    history,          // [{role:'user'|'model', text:'', timestamp:''}]
    model,            // {provider, model, label}
    sending,          // boolean — идёт запрос
  },
  // glossary
  glossaryTerms,      // массив терминов из localStorage
  // backgrounds
  backgrounds,        // [{name, url}] — ObjectURL для фоновых изображений
  bgIndex,            // -1 = выкл, >=0 = индекс активного фона
}
```

---

## Браузерная поддержка

Поддерживается: **Chrome 107+, Edge 107+** (Linux, Windows).
Не поддерживается: Firefox, Safari, мобильные.

---

## Интернационализация (i18n)

Словари в `translations.js`, функции:
- `t(key)` — перевод с fallback-цепью `LANG[_lang][key] → LANG.en[key] → key`
- `setLang(lang)` — переключает язык, сохраняет в `localStorage('meetscribe_lang')`
- `getCurrentLang()` — текущий язык (`'en'` | `'ru'`)

**Правила:**
- Все пользовательские строки в JS через `t('key')`, **не инлайновый текст**
- `applyTranslations()` — применяет переводы к статичному HTML, вызывается в `init()` и при смене языка через кнопку `#btn-lang`
- Параметры в строках: `.replace('{param}', value)` — например `t('ai_processing').replace('{s}', seconds)`
- Даты/время в UI: `getCurrentLang() === 'ru' ? 'ru-RU' : 'en-US'` для `toLocaleDateString/toLocaleTimeString`
- AI-промты (rename, summarize, chat system prompt) **намеренно оставлены на русском** — пользователь хочет русский вывод от AI
- Имена переменных: **не называть переменные `t`** (конфликт с функцией `t()`); использовать `errBody`, `toast`, `respText` и т.п.

---

## Что не делать

- Не разбивать `index.html` на модули (исключение: `translations.js` — уже вынесен)
- Не добавлять фреймворки (React, Vue, ...) и сборщики (Webpack, Vite, ...)
- Не использовать `@google/generative-ai` SDK для Live API — только нативный WebSocket
- Не хранить транскрипции в localStorage или облаке
- Не открывать `file://` для тестирования
- Не менять модель Gemini без проверки доступности через ListModels
- Не размещать `inputAudioTranscription` внутри `generationConfig` — только на уровне `setup`
- Не слать `{ clientContent: { turnComplete: true } }` для принудительного VAD — Gemini отклоняет
- Не использовать параметр `keywords` для Deepgram nova-3 — только `keyterm`
- Не называть переменные `t` — конфликт с функцией `t()` из translations.js
