# 🌌 Interactive 3D Solar System — ActionScript Project

This project is an **interactive 3D-style Solar System simulation** built with **ActionScript 3.0**.  
It visualizes planetary motion, rotation, and real-time data display with dynamic zoom, smooth camera movement, and depth effects.

---

## ✨ Features

- 🪐 **Elliptical Orbits** — Each planet moves along its calculated orbital path.  
- 🌞 **Cinematic Sun Glow** — Realistic glow effect using `BlurFilter` for a soft halo.  
- 🧭 **Interactive Camera**  
  - Drag to rotate the view  
  - Scroll to zoom or pan vertically  
- 🔍 **Object Zoom & Info Panel**  
  - Click a planet or the Sun to zoom in  
  - Displays temperature, key characteristics, and Wikipedia link  
- 🌫️ **Depth Fog Effect** — Adds atmospheric perspective for distant planets.  
- 💫 **Smooth Animations** — Uses linear interpolation for zoom and movement transitions.  
- 📚 **Educational Content** — Each celestial body includes curated factual data.

---

## 🧩 Technologies Used

| Component | Description |
|------------|-------------|
| **ActionScript 3.0** | Main programming language |
| **Adobe Animate / Flash Pro** | Stage setup and movie clip linkage |
| **Flash Display API** | Used for rendering shapes, layers, and camera |
| **URLRequest / navigateToURL** | For linking to external Wikipedia pages |
| **BlurFilter** | Creates cinematic glow around the Sun |

---

## 🪐 Object Details

| Object | Avg. Temperature | Key Characteristic | Source |
|:-------|:----------------:|:------------------|:-------|
| **Sun** | +5,500 °C | Central G-type star; 99.86% of total system mass | [Wikipedia](https://id.wikipedia.org/wiki/Matahari) |
| **Mercury** | +167 °C | Smallest and closest planet; extreme temperature swings | [Wikipedia](https://id.wikipedia.org/wiki/Merkurius_(planet)) |
| **Venus** | +467 °C | Hottest planet; extreme greenhouse effect | [Wikipedia](https://id.wikipedia.org/wiki/Venus) |
| **Earth** | +15 °C | Blue planet with life and abundant water | [Wikipedia](https://id.wikipedia.org/wiki/Bumi) |
| **Mars** | −63 °C | The Red Planet; possible ice deposits | [Wikipedia](https://id.wikipedia.org/wiki/Mars) |

---

## 🕹️ Controls

| Action | Description |
|:-------|:-------------|
| **Left-click on planet** | Zoom in and show information panel |
| **Click again** | Zoom out and return to overview |
| **Mouse drag** | Rotate or pan camera |
| **Mouse wheel** | Zoom in/out or vertical movement (with *Shift* key) |

---

## 🧠 Code Highlights

### 🔸 Custom Info Panel with Shape Background
```actionscript
function drawCustomInfoBackground(w:Number, h:Number, color:uint, alpha:Number, padding:Number = 10):void {
    infoBackground.graphics.clear();
    infoBackground.graphics.beginFill(0x000000, 0.60);
    infoBackground.graphics.drawRoundRect(-padding, -padding, w + padding*2, h + padding*2, 15);
    infoBackground.graphics.endFill();
}
