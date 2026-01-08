<!doctype html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Leitor de QR (Modo Hitster)</title>
  <style>
    body { font-family: system-ui, Arial; margin: 24px; }
    .card { border: 1px solid #ddd; border-radius: 12px; padding: 16px; max-width: 520px; }
    button { padding: 12px 14px; border-radius: 10px; border: 1px solid #ccc; background: #fff; cursor: pointer; }
    #reader { margin-top: 12px; }
    .muted { color: #666; }
    .row { display:flex; gap:10px; flex-wrap: wrap; margin-top: 10px; }
    code { background:#f6f6f6; padding:2px 6px; border-radius:6px; }
  </style>
  <!-- Leitor de QR via biblioteca estável -->
  <script src="https://unpkg.com/html5-qrcode"></script>
</head>
<body>
  <div class="card">
    <h2>Leitor de QR — “vira pra tocar”</h2>
    <p class="muted">1) Escaneie o QR. 2) Vire o celular com a tela para baixo para abrir no Spotify.</p>

    <div class="row">
      <button id="btnSensors">Ativar sensores</button>
      <button id="btnScan">Escanear QR</button>
      <button id="btnPlay" disabled>Tocar agora (fallback)</button>
      <button id="btnReset">Reset</button>
    </div>

    <p id="status" style="margin-top:12px;">Status: aguardando…</p>

    <div id="reader" style="width:320px;"></div>
    <p class="muted">Dica: no iPhone, você precisa tocar em <b>Ativar sensores</b> e permitir acesso.</p>
  </div>

<script>
  let pendingUrl = null;
  let armed = false;
  let scanner = null;

  const statusEl = document.getElementById("status");
  const btnSensors = document.getElementById("btnSensors");
  const btnScan = document.getElementById("btnScan");
  const btnPlay = document.getElementById("btnPlay");
  const btnReset = document.getElementById("btnReset");

  function setStatus(msg) { statusEl.textContent = "Status: " + msg; }

  function openSpotify(url) {
    setStatus("abrindo no Spotify…");
    window.location.href = url;
  }

  function onQrDecoded(decodedText) {
    pendingUrl = decodedText.trim();
    armed = true;
    btnPlay.disabled = false;
    setStatus("QR lido. Vire o celular para baixo para tocar.");
  }

  function isFaceDown(acc) {
    // acc.z ~ -9.8 quando tela está para baixo (varia por aparelho)
    return acc && typeof acc.z === "number" && acc.z < -7;
  }

  function handleMotion(e) {
    if (!armed || !pendingUrl) return;
    const acc = e.accelerationIncludingGravity;
    if (isFaceDown(acc)) {
      armed = false;
      openSpotify(pendingUrl);
    }
  }

  async function enableSensors() {
    try {
      if (typeof DeviceMotionEvent !== "undefined" &&
          typeof DeviceMotionEvent.requestPermission === "function") {
        const res = await DeviceMotionEvent.requestPermission();
        if (res !== "granted") {
          setStatus("permissão de sensores negada. Use 'Tocar agora' (fallback).");
          return;
        }
      }
      window.addEventListener("devicemotion", handleMotion, { passive: true });
      setStatus("sensores ativados. Agora escaneie um QR.");
    } catch (err) {
      setStatus("não consegui ativar sensores. Use 'Tocar agora' (fallback).");
    }
  }

  async function startScan() {
    if (!scanner) scanner = new Html5Qrcode("reader");
    setStatus("abrindo câmera…");
    try {
      const cams = await Html5Qrcode.getCameras();
      if (!cams || !cams.length) { setStatus("nenhuma câmera encontrada."); return; }
      const camId = cams[0].id;

      await scanner.start(
        { deviceId: { exact: camId } },
        { fps: 10, qrbox: 250 },
        (decodedText) => {
          // para de escanear assim que lê
          scanner.stop().catch(()=>{});
          onQrDecoded(decodedText);
        }
      );
      setStatus("aponte para o QR code…");
    } catch (err) {
      setStatus("falha ao abrir câmera. Verifique permissões/HTTPS.");
    }
  }

  function resetAll() {
    pendingUrl = null;
    armed = false;
    btnPlay.disabled = true;
    setStatus("aguardando…");
    if (scanner) scanner.stop().catch(()=>{});
    document.getElementById("reader").innerHTML = "";
    scanner = null;
  }

  btnSensors.addEventListener("click", enableSensors);
  btnScan.addEventListener("click", startScan);
  btnPlay.addEventListener("click", () => pendingUrl && openSpotify(pendingUrl));
  btnReset.addEventListener("click", resetAll);

  setStatus("toque em 'Ativar sensores' e depois 'Escanear QR'.");
</script>
</body>
</html>
