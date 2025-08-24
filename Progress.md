```dataviewjs
const container = this.container;

// === Config ===
const ip = "10.0.78.108:7860"; // change this if needed
const baseUrl = `http://${ip}/sdapi/v1`;

const endpoint = `${baseUrl}/progress`;
const interruptUrl = `${baseUrl}/interrupt`;
const skipUrl = `${baseUrl}/skip`;
const updateInterval = 1000; // ms

// UI Elements
let status = document.createElement("div");
status.style.fontSize = "16px";
status.style.fontFamily = "monospace";
status.innerText = "⏳ Fetching progress...";

let image = document.createElement("img");
image.style.maxWidth = "100%";
image.style.marginTop = "1em";

// Buttons
let buttonContainer = document.createElement("div");
buttonContainer.style.marginTop = "1em";
buttonContainer.style.display = "flex";
buttonContainer.style.gap = "1em";

let interruptBtn = document.createElement("button");
interruptBtn.innerText = "🛑 Interrupt";
interruptBtn.onclick = async () => {
  try {
    await fetch(interruptUrl, { method: "POST" });
    status.innerText = "🔴 Interrupt sent.";
  } catch (e) {
    status.innerText = "❌ Failed to send interrupt: " + e;
  }
};

let skipBtn = document.createElement("button");
skipBtn.innerText = "⏭️ Skip";
skipBtn.onclick = async () => {
  try {
    await fetch(skipUrl, { method: "POST" });
    status.innerText = "⏭️ Skip sent.";
  } catch (e) {
    status.innerText = "❌ Failed to send skip: " + e;
  }
};

buttonContainer.appendChild(interruptBtn);
buttonContainer.appendChild(skipBtn);

// Add all to container
container.appendChild(status);
container.appendChild(buttonContainer);
container.appendChild(image);

// Update loop
async function updateProgress() {
  try {
    const res = await fetch(endpoint);
    const data = await res.json();

    const pct = Math.round(data.progress * 100);
    const eta = data.eta_relative.toFixed(1);
    const job = data.state.job || "unknown";
    const step = data.state.sampling_step ?? "?";
    const total = data.state.sampling_steps ?? "?";

    status.innerText = `📦 Job: ${job}
🚧 Progress: ${pct}%
🔁 Step: ${step} / ${total}
⏱ ETA: ${eta}s`;

    if (data.current_image) {
      image.src = "data:image/png;base64," + data.current_image;
    }
  } catch (err) {
    status.innerText = "❌ Error fetching progress: " + err;
  }
}

// Start loop
updateProgress();
setInterval(updateProgress, updateInterval);
```
