[index.html](https://github.com/user-attachments/files/34345217/index.html)

<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<title>Stuart IoT | Dashboard</title>

<script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<style>
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  padding: 20px;
  background: #101820;
  color: #ffffff;    
  font-family: Arial, sans-serif;
}

.container {
  max-width: 1100px;
  margin: auto;
}

header {
  margin-bottom: 25px;
}

header h1 {
  margin-bottom: 5px;
  color: #32c5a0;
}

header p {
  color: #aab7c4;
}

.card {
  background: #1c2935;
  border: 1px solid #304150;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 18px;
}

input {
  width: 100%;
  padding: 12px;
  margin: 6px 0 12px;
  border-radius: 7px;
  border: 1px solid #465968;
  background: #101820;
  color: white;
}

button {
  border: none;
  border-radius: 8px;
  padding: 12px 18px;
  cursor: pointer;
  font-weight: bold;
  color: white;
}

.connect {
  background: #087f68;
  width: 100%;
}

.relay {
  background: #43576a;
  width: 100%;
  margin-top: 12px;
}

.relay.on {
  background: #087f68;
}

button:disabled {
  opacity: .4;
  cursor: not-allowed;
}

.status {
  display: inline-block;
  padding: 8px 12px;
  border-radius: 20px;
  background: #594b30;
  color: #ffd36c;
  font-size: 13px;
}

.status.online {
  background: #164d3d;
  color: #63e6b1;
}

.status.offline {
  background: #582e35;
  color: #ff9ba5;
}

.grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 18px;
}

.value {
  font-size: 35px;
  font-weight: bold;
  margin: 12px 0;
  color: #32c5a0;
}

.unit {
  color: #aab7c4;
}

.chart-box {
  height: 240px;
  position: relative;
}

.small {
  font-size: 13px;
  color: #aab7c4;
}

@media(max-width: 650px) {
  body {
    padding: 12px;
  }

  .grid {
    grid-template-columns: 1fr;
  }

  .value {
    font-size: 30px;
  }
}
</style>
</head>

<body>

<div class="container">

<header>
  <h1>STUART IoT</h1>
  <p>Monitoramento e automação inteligente</p>
  <span id="connection" class="status">
    Desconectado
  </span>
</header>

<!-- CONEXAO -->

<section class="card">

<h3>Conexão MQTT</h3>

<label>Hostname HiveMQ</label>
<input id="host"
placeholder="seu-cluster.s1.eu.hivemq.cloud">

<label>Usuário MQTT</label>
<input id="username" autocomplete="off">

<label>Senha MQTT</label>
<input id="password" type="password"
autocomplete="new-password">

<button class="connect" id="connectBtn"
onclick="connectMQTT()">
CONECTAR AO HIVEMQ
</button>

<p class="small" id="message">
Aguardando conexão...
</p>

</section>

<!-- SENSORES -->

<div class="grid">

<section class="card">
<h3>Temperatura</h3>
<div class="value">
<span id="temperature">--</span>
<span class="unit">°C</span>
</div>
<p class="small">Sensor DHT11</p>
</section>

<section class="card">
<h3>Umidade</h3>
<div class="value">
<span id="humidity">--</span>
<span class="unit">%</span>
</div>
<p class="small">Sensor DHT11</p>
</section>

</div>

<!-- RELES -->

<section class="card">

<h3>Controle de saídas</h3>

<div class="grid">

<div>
<h4>Relé 01</h4>
<span id="relay1State" class="status">
Desconhecido
</span>

<button id="relay1Btn" class="relay"
disabled onclick="toggleRelay(1)">
LIGAR
</button>
</div>

<div>
<h4>Relé 02</h4>
<span id="relay2State" class="status">
Desconhecido
</span>

<button id="relay2Btn" class="relay"
disabled onclick="toggleRelay(2)">
LIGAR
</button>
</div>

</div>

</section>

<!-- GRAFICOS -->

<section class="card">
<h3>Histórico de temperatura</h3>
<div class="chart-box">
<canvas id="temperatureChart"></canvas>
</div>
</section>

<section class="card">
<h3>Histórico de umidade</h3>
<div class="chart-box">
<canvas id="humidityChart"></canvas>
</div>
</section>

<section class="card">
<p class="small">
Última atualização:
<strong id="lastUpdate">--</strong>
</p>
<p class="small">
Equipamento: ESP32-S3 - Stuart 001
</p>
</section>

</div>

<script>

// =================================
// CONFIGURACAO MQTT
// =================================

const BASE = "stuart/esp32s3_001/";

const TOPICS = {
  telemetry: BASE + "telemetry",
  availability: BASE + "availability",
  relay1Cmd: BASE + "relay1/cmd",
  relay1State: BASE + "relay1/state",
  relay2Cmd: BASE + "relay2/cmd",
  relay2State: BASE + "relay2/state"
};

let client = null;

let states = {
  relay1: null,
  relay2: null
};

let deviceOnline = false;

// =================================
// ELEMENTOS DA PAGINA
// =================================

const connection = document.getElementById("connection");
const message = document.getElementById("message");

function setConnection(text, type) {
  connection.textContent = text;
  connection.className = "status " + type;
}

function updateRelayButton(number) {

  const state = states["relay" + number];

  const btn = document.getElementById(
    "relay" + number + "Btn"
  );

  const label = document.getElementById(
    "relay" + number + "State"
  );

  btn.disabled = !client?.connected ||
                 !deviceOnline ||
                 state === null;

  btn.textContent = state === true ? "DESLIGAR" : "LIGAR";

  btn.classList.toggle("on", state === true);

  label.textContent =
    state === true ? "LIGADO" :
    state === false ? "DESLIGADO" :
    "DESCONHECIDO";

  label.className = state === true
    ? "status online"
    : state === false
      ? "status offline"
      : "status";
}

function updateControls() {
  updateRelayButton(1);
  updateRelayButton(2);
}

// =================================
// CONECTAR AO HIVEMQ
// =================================

function connectMQTT() {

  if (client) {
    client.end(true);
    client = null;
  }

  const host = document.getElementById("host")
    .value.trim();

  const username = document.getElementById("username")
    .value.trim();

  const password = document.getElementById("password")
    .value;

  if (!host || !username || !password) {
    message.textContent =
      "Preencha todos os campos.";
    return;
  }

  if (!/^[a-zA-Z0-9.-]+$/.test(host)) {
    message.textContent =
      "Hostname inválido.";
    return;
  }

  const url = "wss://" + host + ":8884/mqtt";

  setConnection("Conectando...", "");

  message.textContent = "Iniciando conexão segura...";

  client = mqtt.connect(url, {

    clientId: "stuart_web_" +
      Math.random().toString(16).slice(2, 10),

    username: username,
    password: password,

    clean: true,
    connectTimeout: 10000,
    reconnectPeriod: 3000,
    reconnectOnConnackError: true,
    keepalive: 30

  });

  client.on("connect", function() {

    setConnection("MQTT CONECTADO", "online");

    message.textContent =
      "Conectado ao HiveMQ Cloud.";

    client.subscribe([
      TOPICS.telemetry,
      TOPICS.availability,
      TOPICS.relay1State,
      TOPICS.relay2State
    ], { qos: 0 }, function(err) {

      if (err) {
        message.textContent =
          "Erro ao assinar tópicos.";
      }
    });

  });

  client.on("reconnect", function() {
    setConnection("Reconectando...", "");
  });

  client.on("offline", function() {
    setConnection("Offline", "offline");
  });

  client.on("close", function() {
    setConnection("Desconectado", "offline");
  });

  client.on("error", function(err) {
    message.textContent =
      "Falha MQTT. Verifique conexão e credenciais.";
    console.error(err);
  });

  client.on("message", function(topic, payload) {

    const data = payload.toString();

    // DISPONIBILIDADE

    if (topic === TOPICS.availability) {

      deviceOnline = data === "ONLINE";

      updateControls();

      return;
    }

    // ESTADO DOS RELES

    if (topic === TOPICS.relay1State) {
      if (data === "ON") states.relay1 = true;
      if (data === "OFF") states.relay1 = false;
      updateRelayButton(1);
      return;
    }

    if (topic === TOPICS.relay2State) {
      if (data === "ON") states.relay2 = true;
      if (data === "OFF") states.relay2 = false;
      updateRelayButton(2);
      return;
    }

    // TELEMETRIA

    if (topic === TOPICS.telemetry) {

      try {

        const values = JSON.parse(data);

        const temp = Number(values.temperature);
        const hum = Number(values.humidity);

        if (!Number.isFinite(temp) ||
            !Number.isFinite(hum)) {
          return;
        }

        if (temp < -40 || temp > 80 ||
            hum < 0 || hum > 100) {
          return;
        }

        document.getElementById("temperature")
          .textContent = temp.toFixed(1);

        document.getElementById("humidity")
          .textContent = hum.toFixed(1);

        document.getElementById("lastUpdate")
          .textContent = new Date()
          .toLocaleTimeString("pt-BR");

        addChartData(temp, hum);

      } catch(error) {
        console.error("Telemetria inválida", error);
      }
    }

  });

  updateControls();
}

// =================================
// COMANDOS DOS RELES
// =================================

function toggleRelay(number) {

  if (!client?.connected || !deviceOnline) return;

  const current = states["relay" + number];

  if (current === null) return;

  const next = current ? "OFF" : "ON";

  const topic = number === 1
    ? TOPICS.relay1Cmd
    : TOPICS.relay2Cmd;

  client.publish(
    topic,
    next,
    { qos: 1, retain: false },
    function(err) {
      if (err) {
        message.textContent =
          "Falha ao enviar comando.";
      } else {
        message.textContent =
          "Comando enviado. Aguardando confirmação.";
      }
    }
  );
}

// =================================
// GRAFICOS CHART.JS
// =================================

function createChart(element, label, color) {

  return new Chart(element, {

    type: "line",

    data: {
      labels: [],
      datasets: [{
        label: label,
        data: [],
        borderColor: color,
        backgroundColor: color,
        tension: 0.3,
        pointRadius: 2
      }]
    },

    options: {

      responsive: true,
      maintainAspectRatio: false,

      animation: false,

      scales: {
        x: {
          ticks: { color: "#aab7c4" },
          grid: { color: "#304150" }
        },
        y: {
          ticks: { color: "#aab7c4" },
          grid: { color: "#304150" }
        }
      },

      plugins: {
        legend: {
          labels: { color: "#ffffff" }
        }
      }
    }

  });
}

const tempChart = createChart(
  document.getElementById("temperatureChart"),
  "Temperatura °C",
  "#32c5a0"
);

const humChart = createChart(
  document.getElementById("humidityChart"),
  "Umidade %",
  "#55aaff"
);

function addChartData(temp, hum) {

  const label = new Date()
    .toLocaleTimeString("pt-BR");

  [tempChart, humChart].forEach(chart => {

    chart.data.labels.push(label);

    if (chart.data.labels.length > 20) {
      chart.data.labels.shift();
    }
  });

  tempChart.data.datasets[0].data.push(temp);
  humChart.data.datasets[0].data.push(hum);

  if (tempChart.data.datasets[0].data.length > 20) {
    tempChart.data.datasets[0].data.shift();
    humChart.data.datasets[0].data.shift();
  }

  tempChart.update();
  humChart.update();
}

</script>

</body>
</html>
