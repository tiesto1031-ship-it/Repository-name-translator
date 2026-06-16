<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>中日即時翻譯 (純文字版)</title>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: sans-serif; background: #0d0d1a; color: #e8e8f0; min-height: 100vh; display: flex; flex-direction: column; align-items: center; padding: 30px 16px; }
  h1 { font-size: 1.4rem; color: #c8b8ff; text-align: center; margin-bottom: 4px; }
  .sub { font-size: 0.75rem; color: #4040a0; text-align: center; margin-bottom: 30px; }
  .orb { width: 110px; height: 110px; border-radius: 50%; background: radial-gradient(circle at 35% 35%, #c8b8ff, #6644cc); display: flex; align-items: center; justify-content: center; font-size: 2.4rem; cursor: pointer; border: none; margin-bottom: 14px; transition: transform 0.1s; }
  .orb:active { transform: scale(0.95); }
  .orb.listening { animation: orbPulse 1.4s infinite; background: radial-gradient(circle at 35% 35%, #ff8aaa, #cc2244); }
  .orb.thinking { background: radial-gradient(circle at 35% 35%, #ffcc88, #cc7722); }
  @keyframes orbPulse { 0%, 100% { box-shadow: 0 0 0 0 #ff6b8a66; } 50% { box-shadow: 0 0 0 24px #ff6b8a00; } }
  .status { font-size: 0.85rem; color: #6060b0; text-align: center; min-height: 22px; margin-bottom: 24px; }
  .status.listening { color: #ff8aaa; } .status.thinking { color: #ffcc88; }
  .card { width: 100%; max-width: 440px; background: #1a1a2e; border-radius: 18px; padding: 16px; margin-bottom: 12px; }
  .card-label { font-size: 0.65rem; letter-spacing: 0.15em; text-transform: uppercase; color: #4040a0; margin-bottom: 8px; }
  .text-display { min-height: 80px; font-size: 1.05rem; line-height: 1.7; word-break: break-all; }
  .source { color: #e8e8f0; } .result { color: #c8b8ff; }
  .placeholder { color: #333360; font-style: italic; }
</style>
</head>
<body>
<h1>中日即時翻譯</h1>
<p class="sub">純文字顯示 · 無干擾連續翻譯</p>
<button class="orb" id="orb" onclick="toggleListen()">🎤</button>
<div class="status" id="status">點擊開始，說話後自動翻譯</div>
<div class="card">
  <div class="card-label">偵測到的語音</div>
  <div class="text-display source" id="sourceBox"><span class="placeholder">說話後顯示…</span></div>
</div>
<div class="card">
  <div class="card-label">翻譯結果</div>
  <div class="text-display result" id="resultBox"><span class="placeholder">翻譯結果…</span></div>
</div>
<script>
  let recognition = null, isAutoListen = false, translateTimer = null, finalText = '';
  const orb = document.getElementById('orb'), statusEl = document.getElementById('status'), sourceBox = document.getElementById('sourceBox'), resultBox = document.getElementById('resultBox');

  function toggleListen() {
    if (isAutoListen) {
      isAutoListen = false;
      if(recognition) recognition.stop();
      orb.classList.remove('listening');
      orb.textContent = '🎤';
      setStatus('已停止', '');
    } else {
      isAutoListen = true;
      startListen();
    }
  }

  function startListen() {
    const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
    if(!SR) { setStatus('❌ 請用 Chrome', ''); return; }

    finalText = '';
    sourceBox.innerHTML = '<span class="placeholder">收音中，請說話...</span>';
    
    recognition = new SR();
    recognition.lang = 'zh-TW';
    recognition.continuous = true; 
    recognition.interimResults = true;

    recognition.onstart = () => {
      orb.classList.add('listening');
      orb.classList.remove('thinking');
      orb.textContent = '⏹';
      setStatus('🎤 收音中…', 'listening');
    };

    recognition.onresult = (e) => {
      let interim = '';
      let currentFinal = '';
      
      for(let i = e.resultIndex; i < e.results.length; i++) {
        if(e.results[i].isFinal) currentFinal += e.results[i][0].transcript;
        else interim += e.results[i][0].transcript;
      }
      
      if(currentFinal) finalText += currentFinal;
      
      sourceBox.innerHTML = finalText + (interim ? `<span style="color:#6060a0">${interim}</span>` : '');

      clearTimeout(translateTimer);
      
      let textToTranslate = (finalText + interim).trim();
      if(textToTranslate) {
        // 停頓 1.2 秒沒說話，就送出翻譯
        translateTimer = setTimeout(() => {
          doTranslate(textToTranslate);
          finalText = ''; // 翻譯後清空畫面上的原文，準備聽下一句
        }, 1200);
      }
    };

    recognition.onerror = (e) => {
      if(e.error === 'no-speech' || e.error === 'aborted') return;
    };

    recognition.onend = () => {
      // 只有在系統因為太久沒講話自己斷掉時，才會安靜重啟，不會再閃爍
      if (isAutoListen) {
        setTimeout(() => {
          if (isAutoListen) startListen();
        }, 300); 
      }
    };

    try { recognition.start(); } catch(err) {}
  }

  async function doTranslate(text) {
    orb.classList.add('thinking');
    setStatus('翻譯中…', 'thinking');

    // 注意：這裡移除了 recognition.abort()，麥克風不會再被切斷了！

    const isJa = /[\u3040-\u30ff]/.test(text);
    const sl = isJa ? 'ja' : 'zh-TW';
    const tl = isJa ? 'zh-TW' : 'ja';

    try {
      const res = await fetch(`https://api.mymemory.translated.net/get?q=${encodeURIComponent(text)}&langpair=${sl}|${tl}`);
      const data = await res.json();
      const t = data.responseData?.translatedText || '';
      if(!t) throw new Error();

      resultBox.textContent = t;
      
    } catch {
      resultBox.textContent = '❌ 翻譯失敗，請重試';
    } finally {
      // 翻譯完成後恢復 UI 狀態
      if (isAutoListen) {
        orb.classList.remove('thinking');
        setStatus('🎤 收音中…', 'listening');
      }
    }
  }
</script>
</body>
</html>
