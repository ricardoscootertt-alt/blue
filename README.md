<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Conector Bluetooth Web</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            background-color: #0f172a;
            color: #f8fafc;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        .glass {
            background: rgba(30, 41, 59, 0.7);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.1);
        }
        .status-dot {
            height: 10px;
            width: 10px;
            border-radius: 50%;
            display: inline-block;
        }
        .pulse {
            animation: pulse-animation 2s infinite;
        }
        @keyframes pulse-animation {
            0% { box-shadow: 0 0 0 0px rgba(59, 130, 246, 0.5); }
            100% { box-shadow: 0 0 0 10px rgba(59, 130, 246, 0); }
        }
    </style>
</head>
<body class="min-h-screen flex flex-col items-center justify-center p-4">

    <div class="max-w-md w-full glass rounded-3xl p-8 shadow-2xl">
        <div class="flex flex-col items-center mb-8">
            <div id="bt-icon-container" class="bg-blue-600 p-4 rounded-full mb-4 shadow-lg">
                <svg xmlns="http://www.w3.org/2000/svg" class="h-8 w-8 text-white" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 7l10 10M7 17L17 7V17H7V7z" />
                    <path d="M6.5 16.5L17.5 5.5V18.5L6.5 7.5" stroke="white" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
            </div>
            <h1 class="text-2xl font-bold text-center">Conector Bluetooth</h1>
            <p class="text-slate-400 text-sm mt-2 text-center">Interface direta para Windows 10 via Web API</p>
        </div>

        <!-- Alerta de Segurança/Iframe -->
        <div id="iframe-warning" class="hidden bg-amber-900/50 border border-amber-500 text-amber-200 p-4 rounded-xl mb-6 text-xs">
            <p class="font-bold mb-1">Restrição de Segurança Detectada:</p>
            O Bluetooth não pode ser iniciado dentro de um frame. Para usar, salve este código como um arquivo <b>.html</b> e abra-o diretamente no Chrome ou Edge.
        </div>

        <div id="browser-warning" class="hidden bg-red-900/50 border border-red-500 text-red-200 p-3 rounded-xl mb-6 text-sm">
            Seu navegador não suporta a Web Bluetooth API.
        </div>

        <div class="space-y-4">
            <button id="scan-btn" class="w-full bg-blue-600 hover:bg-blue-500 transition-colors py-3 rounded-xl font-semibold flex items-center justify-center gap-2 shadow-lg active:scale-95 transform">
                <span>Procurar Dispositivos</span>
            </button>

            <div id="device-info" class="hidden space-y-3 pt-4 border-t border-white/10">
                <div class="flex justify-between items-center">
                    <span class="text-slate-400">Dispositivo Atual:</span>
                    <span id="device-name" class="font-mono text-blue-400">Nenhum</span>
                </div>
                <div class="flex justify-between items-center">
                    <span class="text-slate-400">Status:</span>
                    <div class="flex items-center gap-2">
                        <span id="status-text">Desconectado</span>
                        <span id="status-indicator" class="status-dot bg-slate-500"></span>
                    </div>
                </div>
                <button id="disconnect-btn" class="hidden w-full border border-red-500/50 hover:bg-red-500/20 text-red-400 py-2 rounded-lg text-sm transition-all">
                    Desconectar
                </button>
            </div>
        </div>

        <div class="mt-8">
            <h2 class="text-xs font-bold uppercase tracking-widest text-slate-500 mb-2">Console de Log</h2>
            <div id="log-container" class="bg-black/40 rounded-xl p-3 h-32 overflow-y-auto text-[10px] font-mono text-green-400 border border-white/5">
                <div>> Pronto para escanear...</div>
            </div>
        </div>
    </div>

    <script>
        const scanBtn = document.getElementById('scan-btn');
        const disconnectBtn = document.getElementById('disconnect-btn');
        const logContainer = document.getElementById('log-container');
        const deviceName = document.getElementById('device-name');
        const statusText = document.getElementById('status-text');
        const statusIndicator = document.getElementById('status-indicator');
        const deviceInfo = document.getElementById('device-info');
        const btIconContainer = document.getElementById('bt-icon-container');
        const iframeWarning = document.getElementById('iframe-warning');

        let connectedDevice = null;

        function log(message) {
            const entry = document.createElement('div');
            entry.textContent = `> ${message}`;
            logContainer.appendChild(entry);
            logContainer.scrollTop = logContainer.scrollHeight;
        }

        // Verifica se está rodando em um iframe
        const isIframe = window.self !== window.top;

        if (!navigator.bluetooth) {
            document.getElementById('browser-warning').classList.remove('hidden');
            scanBtn.disabled = true;
            scanBtn.classList.add('opacity-50', 'cursor-not-allowed');
            log('ERRO: API de Bluetooth indisponível neste navegador.');
        }

        async function requestBluetoothDevice() {
            try {
                log('Iniciando busca de dispositivos...');
                
                const device = await navigator.bluetooth.requestDevice({
                    acceptAllDevices: true,
                    optionalServices: ['battery_service', 'heart_rate']
                });

                connectedDevice = device;
                deviceName.textContent = device.name || `ID: ${device.id.substring(0, 8)}...`;
                deviceInfo.classList.remove('hidden');
                iframeWarning.classList.add('hidden');
                
                log(`Dispositivo selecionado: ${device.name}`);
                device.addEventListener('gattserverdisconnected', onDisconnected);
                connect();
            } catch (error) {
                if (error.name === 'SecurityError') {
                    log('ERRO DE SEGURANÇA: O Bluetooth foi bloqueado por estar dentro de um frame.');
                    iframeWarning.classList.remove('hidden');
                } else {
                    log(`Erro: ${error.message}`);
                }
                console.error(error);
            }
        }

        async function connect() {
            if (!connectedDevice) return;
            try {
                log('Tentando conectar ao servidor GATT...');
                statusText.textContent = 'Conectando...';
                statusIndicator.className = 'status-dot bg-yellow-500 pulse';
                const server = await connectedDevice.gatt.connect();
                log('Conexão estabelecida!');
                statusText.textContent = 'Conectado';
                statusIndicator.className = 'status-dot bg-blue-500';
                btIconContainer.classList.add('pulse');
                disconnectBtn.classList.remove('hidden');
            } catch (error) {
                log(`Falha na conexão: ${error.message}`);
                onDisconnected();
            }
        }

        function onDisconnected() {
            log('Dispositivo desconectado.');
            statusText.textContent = 'Desconectado';
            statusIndicator.className = 'status-dot bg-slate-500';
            btIconContainer.classList.remove('pulse');
            disconnectBtn.classList.add('hidden');
            connectedDevice = null;
        }

        function disconnect() {
            if (connectedDevice && connectedDevice.gatt.connected) {
                connectedDevice.gatt.disconnect();
            } else {
                onDisconnected();
            }
        }

        scanBtn.addEventListener('click', requestBluetoothDevice);
        disconnectBtn.addEventListener('click', disconnect);
    </script>
</body>
</html>
