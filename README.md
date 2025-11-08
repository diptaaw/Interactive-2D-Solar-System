stop();

import flash.events.MouseEvent;
import flash.events.Event;
import flash.events.TextEvent;
import flash.display.Shape;
import flash.display.DisplayObject;
import flash.display.Sprite;
import flash.text.TextField;
import flash.text.TextFormat;
import flash.display.MovieClip;
import flash.geom.Point;
import flash.text.TextFormatAlign;
import flash.net.URLRequest;
import flash.net.navigateToURL;
import flash.filters.BlurFilter; // Diperlukan untuk efek Glow yang halus

// ===============================================
// 1. DEKLARASI VARIABEL GLOBAL & KONFIGURASI
// ===============================================
var centerX:Number = stage.stageWidth / 2;
var centerY:Number = stage.stageHeight / 2;
var scaleSecPerDay:Number = 60 / 365.25; 

// Kamera
var camAngle:Number = 0;
var camTilt:Number = 0.4;
var camZoom:Number = 800;
var isDragging:Boolean = false;
var lastMouseX:Number = 0;
var lastMouseY:Number = 0;

// Variabel Zoom Interaktif
var isZoomed:Boolean = false;
var targetZoom:Number = camZoom;
var targetXOffset:Number = 0;
var targetYOffset:Number = 0;
var zoomedObject:Object = null;
var ZOOM_FACTOR:Number = 3;
var LERP_RATE:Number = 0.2; 

// Variabel Panning Kamera Manual
var camXOffset:Number = 0;
var camYOffset:Number = 0;

// Variabel Kontrol Fade Orbit (FADE LEBIH HALUS)
var orbitFadeSpeed:Number = 1.0; 
var orbitMinAlpha:Number = 0.3; 
var orbitMaxAlpha:Number = 0.9; 

// Variabel Efek Kedalaman >ERROR
var DEPTH_FOG_START:Number = 200; // Jarak Z mulai kabur
var DEPTH_FOG_END:Number = 800;   // Jarak Z kabur maksimal (Maksimal Alpha 50% hilang)

// Konstanta Warna Teks Kontras
var DARK_BACKGROUND_COLOR:uint = 0x111111; 
var LIGHT_TEXT_COLOR:uint = 0xFFFFFF; 
var LIGHT_COLORS:Array = [0xFFD700, 0x7CFC00, 0x00BFFF]; 

// ----- Layers -----
var sunGlowLayer:Shape = new Shape(); // LAYER BARU untuk Glow Matahari
addChild(sunGlowLayer); 

var orbitLayer:Shape = new Shape();
addChild(orbitLayer);

var planetLayer:Sprite = new Sprite();
addChild(planetLayer);

var uiLayer:Sprite = new Sprite();
addChild(uiLayer);

// Background Shape Kustom untuk Panel Informasi
var infoBackground:Shape = new Shape(); 
uiLayer.addChild(infoBackground);

// Panel Informasi (UI Layer)
var infoPanel:TextField = new TextField();
var infoFormat:TextFormat = new TextFormat("_sans", 16, LIGHT_TEXT_COLOR);
infoFormat.align = TextFormatAlign.LEFT; 
infoPanel.defaultTextFormat = infoFormat;
infoPanel.selectable = false;
infoPanel.multiline = true;
infoPanel.wordWrap = true;
infoPanel.width = 280; 
infoPanel.autoSize = "left"; 
infoPanel.visible = false;
var status:Boolean = true;
var s:String = "The status is: " + status.toString();
uiLayer.addChild(infoPanel);

// FAKTA PLANET (dengan URL Wikipedia)
var facts:Object = {
    "Sun": {
        "Suhu Rata-rata": "+5.500 °C (Permukaan)",
        "Karakteristik Kunci": "Bintang G-tipe di pusat Tata Surya, mengandung 99,86% massa total sistem. Menghasilkan energi masif melalui proses fusi nuklir di intinya.",
        "URL": "https://id.wikipedia.org/wiki/Matahari" 
    },
    "Mercury": {
        "Suhu Rata-rata": "+167 °C (Sangat Ekstrem)",
        "Karakteristik Kunci": "Planet terkecil dan terdekat dengan Matahari. Permukaan penuh kawah, mengalami fluktuasi suhu paling ekstrem karena atmosfernya yang sangat tipis.",
        "URL": "https://id.wikipedia.org/wiki/Merkurius_(planet)" 
    },
    "Venus": {
        "Suhu Rata-rata": "+467 °C",
        "Karakteristik Kunci": "Planet terpanas di Tata Surya akibat efek rumah kaca yang ekstrem. Berputar sangat lambat dan mundur (retrogad). Dijuluki 'Bintang Fajar/Senja'.",
        "URL": "https://id.wikipedia.org/wiki/Venus" 
    },
    "Earth": {
        "Suhu Rata-rata": "+15 °C",
        "Karakteristik Kunci": "Planet biru, satu-satunya tempat yang diketahui memiliki kehidupan. Dilindungi oleh atmosfer kaya oksigen dan memiliki air cair yang melimpah di permukaannya.",
        "URL": "https://id.wikipedia.org/wiki/Bumi" 
    },
    "Mars": {
        "Suhu Rata-rata": "-63 °C",
        "Karakteristik Kunci": "Dijuluki Planet Merah karena kandungan besi oksida pada permukaannya. Memiliki gunung tertinggi di Tata Surya, Olympus Mons, serta potensi air es.",
        "URL": "https://id.wikipedia.org/wiki/Mars" 
    }
};

// Planet data
var planets:Array = [
    {name:"Mercury", mcName:"mercury_mc", days:87.97, rot_days:58.6, a:240, b:180, inc:0.05, color:0xFFD700, baseScale:0.40, RadiusPx: 240}, 
    {name:"Venus",   mcName:"venus_mc",  days:224.70, rot_days:243.0, a:360, b:280, inc:0.03, color:0x7CFC00, baseScale:0.50, RadiusPx: 360},
    {name:"Earth",   mcName:"earth_mc",  days:365.25, rot_days:1.0, a:500, b:400, inc:0.02, color:0x00BFFF, baseScale:0.60, RadiusPx: 500},
    {name:"Mars",    mcName:"mars_mc",   days:686.98, rot_days:1.03, a:640, b:500, inc:0.07, color:0xFF4500, baseScale:0.55, RadiusPx: 640}
];

// Sun data
var sunData:Object = {
    name: "Sun", 
    mcName: "sun_mc",
    color: 0xFFA500, 
    mc: null, 
    baseScale: 0.7, 
    rot_days: 27.0, 
    RadiusPx: 0,
    depthZ: 0,
    label: null
};

// VARIABEL BARU UNTUK SMOOTH SCALE MATAHARI
var sunCurrentScale:Number = sunData.baseScale;
var sunTargetScale:Number = sunData.baseScale;

// INFO PANEL STYLE
function drawCustomInfoBackground(w:Number, h:Number, color:uint, alpha:Number, padding:Number = 10):void {
    infoBackground.graphics.clear();
    
    var borderW:Number = w + padding * 2;
    var borderH:Number = h + padding * 2;
    var cornerSize:Number = 15; 

    infoBackground.graphics.beginFill(0x000000, 0.60); 

    infoBackground.graphics.moveTo(cornerSize, 0); 
    infoBackground.graphics.lineTo(borderW, 0);
    infoBackground.graphics.lineTo(borderW - cornerSize, borderH); 
    infoBackground.graphics.lineTo(0, borderH);
    infoBackground.graphics.lineTo(cornerSize, 0); 
    
    infoBackground.graphics.endFill();

    infoBackground.graphics.lineStyle(2, 0xFFFFFF, 0.9);
    infoBackground.graphics.moveTo(cornerSize, 0);
    infoBackground.graphics.lineTo(borderW, 0);
    infoBackground.graphics.lineTo(borderW - cornerSize, borderH);
    infoBackground.graphics.lineTo(0, borderH);
    infoBackground.graphics.lineTo(cornerSize, 0);
}

// 3. FUNGSI HANDLER KLIK (Tidak Berubah)

function handleLink(e:TextEvent):void {
    var request:URLRequest = new URLRequest(e.text);
    try {
        navigateToURL(request, "_blank"); 
    } catch (err:Error) {
        trace("Error membuka URL: " + err.message);
    }
}

function handleClick(e:MouseEvent):void {
    var clickedMC:MovieClip = e.currentTarget as MovieClip;
    var data:Object = clickedMC.data;
    
    var detailFact:Object = facts[data.name];
    var infoText:String = "";
    
    if (detailFact) {
        infoText += "Objek: " + data.name + "\n";
        infoText += "=============================\n"; 
        infoText += "Suhu Rata-rata: " + detailFact["Suhu Rata-rata"] + "\n\n";
        infoText += "Karakteristik Kunci:\n" + detailFact["Karakteristik Kunci"] + "\n\n";
        
        if (detailFact["URL"]) {
             infoText += "<font color='#FFD700'><b>Sumber Wikipedia:</b></font> <a href='" + detailFact["URL"] + "'>" + data.name + "</a>";
        }
        
    } else {
        infoText = "Objek: " + data.name + "\n\nData fakta tidak ditemukan.";
    }


    if (!isZoomed) {
        isZoomed = true;
        zoomedObject = data;
        
        if (data === sunData) {
             targetZoom = camZoom * 1.5; 
             sunTargetScale = sunData.baseScale * 1.5; 
        } else {
             targetZoom = camZoom * ZOOM_FACTOR; 
        }

        infoPanel.backgroundColor = DARK_BACKGROUND_COLOR; 
        
        var tf:TextFormat = infoPanel.getTextFormat();
        tf.color = LIGHT_TEXT_COLOR;
        infoPanel.setTextFormat(tf);
        
        infoPanel.htmlText = infoText;
        infoPanel.visible = true;
        
        var padding:Number = 15;
        drawCustomInfoBackground(infoPanel.textWidth + 20, infoPanel.textHeight + 10, 0x000000, 0.6, padding);
        infoBackground.visible = true;

        infoPanel.addEventListener(TextEvent.LINK, handleLink);
        
        for (var i:int = 0; i < planets.length; i++) {
            if(planets[i].mc) planets[i].label.visible = false;
        }
        if(sunData.mc) sunData.label.visible = false;

    } else if (data === zoomedObject) {
        
        infoPanel.removeEventListener(TextEvent.LINK, handleLink);
        
        isZoomed = false;
        zoomedObject = null;
        
        targetZoom = 800; 
        targetXOffset = 0; 
        targetYOffset = 0;
        
        if (data === sunData) {
            sunTargetScale = sunData.baseScale;
        }

        camXOffset = 0;
        camYOffset = 0;

        infoPanel.visible = false;
        infoBackground.visible = false; 

    } else {
        isZoomed = false; 
        camXOffset = 0;
        camYOffset = 0;
        
        infoPanel.removeEventListener(TextEvent.LINK, handleLink);
        
        handleClick(e);
    }
}

// 4. INISIALISASI PLANET DAN MATAHARI

// 1. Inisialisasi Matahari
if (this[sunData.mcName]) {
    sunData.mc = this[sunData.mcName] as MovieClip;
    sunData.mc.scaleX = sunData.mc.scaleY = sunCurrentScale; 
    sunData.rot_deg_per_sec = (360 / sunData.rot_days) * scaleSecPerDay;
    planetLayer.addChild(sunData.mc);
    sunData.mc.buttonMode = true;
    sunData.mc.mouseChildren = false;
    sunData.mc.data = sunData;
    sunData.mc.addEventListener(MouseEvent.CLICK, handleClick);
    var sunTf:TextField = new TextField();
    sunTf.defaultTextFormat = new TextFormat("_sans", 12, 0xFFFF33, true);
    sunTf.text = sunData.name + "\nOrbit: 0.00 px/s\nRotasi: " + sunData.rot_deg_per_sec.toFixed(4) + " deg/s"; 
    sunTf.width = 120;
    sunTf.height = 45; 
    sunTf.selectable = false;
    uiLayer.addChild(sunTf);
    sunData.label = sunTf;
} else {
    trace("Error: MovieClip for Sun (" + sunData.mcName + ") is null. Check Library Linkage!");
}

// 2. Inisialisasi Planet
var totalPlanets:int = planets.length;
for (var i:int=0; i<totalPlanets; i++) {
    var p:Object = planets[i];
    var mcObj:DisplayObject = this[p.mcName] as DisplayObject;
    if (mcObj) {
        p.mc = mcObj;
        p.mc.scaleX = p.mc.scaleY = 1; 
        planetLayer.addChild(p.mc);
        p.periodSec = p.days * scaleSecPerDay;
        p.omega = 2*Math.PI / p.periodSec;
        p.rot_deg_per_sec = (360 / p.rot_days) * scaleSecPerDay;
        p.angle = Math.random()*Math.PI*2;
        p.mc.buttonMode = true;
        p.mc.mouseChildren = false;
        p.mc.data = p;
        p.mc.addEventListener(MouseEvent.CLICK, handleClick);
        p.xCam = 0;
        p.yCam = 0;
        
        // Offset untuk efek "Loading Circle"
        p.fadeOffset = (i / totalPlanets) * 2 * Math.PI; 

        var tf:TextField = new TextField();
        // Menggunakan font yang sama dengan infoPanel untuk konsistensi visual
        tf.defaultTextFormat = new TextFormat("_sans", 12, LIGHT_TEXT_COLOR, true);
        tf.text = p.name + "\nOrbit: 0.00 px/s\nRotasi: " + p.rot_deg_per_sec.toFixed(4) + " deg/s"; 
        tf.width = 120;
        tf.height = 45; 
        tf.selectable = false;
        uiLayer.addChild(tf);
        p.label = tf;
    } else {
        trace("Error: MovieClip for " + p.name + " (" + p.mcName + ") is null. Check Library Linkage!");
    }
}

// 5. MOUSE HANDLERS & LOOP UTAMA

var elapsedSec:Number = 0;
addEventListener(Event.ENTER_FRAME, tick);

// Mouse controls (Tidak Berubah)
stage.addEventListener(MouseEvent.MOUSE_DOWN, stageMouseDown);
stage.addEventListener(MouseEvent.MOUSE_UP, stageMouseUp);
stage.addEventListener(MouseEvent.MOUSE_MOVE, stageMouseMove);
stage.addEventListener(MouseEvent.MOUSE_WHEEL, stageMouseWheel);

function stageMouseDown(e:MouseEvent):void {
    isDragging = true;
    lastMouseX = mouseX;
    lastMouseY = mouseY;
}
function stageMouseUp(e:MouseEvent):void {
    isDragging = false;
}
function stageMouseMove(e:MouseEvent):void {
    if (isDragging) {
        camAngle += (mouseX - lastMouseX) * 0.01;
        camTilt  -= (mouseY - lastMouseY) * 0.01;
        camTilt = Math.max(-1.2, Math.min(1.2, camTilt));
        camXOffset += (mouseX - lastMouseX);
        camYOffset += (mouseY - lastMouseY);
        lastMouseX = mouseX;
        lastMouseY = mouseY;
    }
}

function stageMouseWheel(e:MouseEvent):void {
    if (e.shiftKey) { 
        camZoom -= e.delta * 80;
        camZoom = Math.max(100, Math.min(5000, camZoom));
        targetZoom = camZoom; 
    } else {
        camYOffset -= e.delta * 10; 
        camYOffset = Math.max(-1000, Math.min(1000, camYOffset));
    }
}

// ----- Main loop -----
function tick(e:Event):void {
    
    // INTERPOLASI (SMOOTH MOVEMENT)
    camZoom += (targetZoom - camZoom) * LERP_RATE;
    
    // ... (Logika Offset & Interpolasi Pusat)
    if (isZoomed && zoomedObject) {
        var targetMC:MovieClip = zoomedObject.mc as MovieClip;
        
        targetXOffset = -(targetMC.x - stage.stageWidth / 2);
        targetYOffset = -(targetMC.y - stage.stageHeight / 2);

        var padding:Number = 15;
        var panelX:Number = targetMC.x + (targetMC.width * targetMC.scaleX) + 10 + padding;
        var panelY:Number = targetMC.y + (targetMC.height * targetMC.scaleY) / 2 + padding;
        
        if (panelX + infoPanel.width + padding > stage.stageWidth) {
              panelX = targetMC.x - infoPanel.width - 10 - padding; 
        }
        if (panelY + infoPanel.height + padding > stage.stageHeight) {
              panelY = stage.stageHeight - infoPanel.height - 10 - padding; 
        }
        
        infoPanel.x = panelX;
        infoPanel.y = panelY;
        infoBackground.x = infoPanel.x - padding;
        infoBackground.y = infoPanel.y - padding;
        
    } else {
        targetXOffset = 0;
        targetYOffset = 0;
    }
    
    var finalTargetX:Number = (stage.stageWidth / 2) + camXOffset + targetXOffset;
    var finalTargetY:Number = (stage.stageHeight / 2) + camYOffset + targetYOffset;
    
    centerX += (finalTargetX - centerX) * LERP_RATE;
    centerY += (finalTargetY - centerY) * LERP_RATE;
    
    if (!isZoomed && Math.abs(targetXOffset) < 1) targetXOffset = 0;
    if (!isZoomed && Math.abs(targetYOffset) < 1) targetYOffset = 0;
    
    if (!isZoomed && Math.abs(centerX - (stage.stageWidth / 2)) < 1 && Math.abs(camXOffset) < 1) {
        centerX = stage.stageWidth / 2;
        centerY = stage.stageHeight / 2;
        camXOffset = 0;
        camYOffset = 0;
    }
    // ... (Akhir Logika Offset & Interpolasi Pusat)

    var dt:Number = 1/stage.frameRate;
    elapsedSec += dt;
    
    orbitLayer.graphics.clear();
    
    var cosA:Number = Math.cos(camAngle);
    var sinA:Number = Math.sin(camAngle);
    var cosT:Number = Math.cos(camTilt);
    var sinT:Number = Math.sin(camTilt);

    var depthList:Array = [];
    
 
    // Efek Sinematik Sun Glow
 
    sunCurrentScale += (sunTargetScale - sunCurrentScale) * LERP_RATE;
    var sunPersp:Number = camZoom / (camZoom + sunData.depthZ);
    var glowRadius:Number = sunData.mc.width * sunCurrentScale * sunPersp * 2.5; // Radius glow 2.5x
    
    sunGlowLayer.graphics.clear();
    
    // Menggambar lingkaran glow dengan alpha yang sangat rendah dan blur (jika filter didukung)
    sunGlowLayer.graphics.beginFill(sunData.color, 0.08); // Alpha sangat rendah
    sunGlowLayer.graphics.drawCircle(centerX, centerY, glowRadius);
    sunGlowLayer.graphics.endFill();
    
    // Penerapan Filter Blur (Memberikan tampilan sinematik/halus)
    // NOTE: Filter Blur harus didukung oleh runtime/environment.
    sunGlowLayer.filters = [new BlurFilter(glowRadius * 0.2, glowRadius * 0.2, 2)]; // Blur 20% dari radius

    // Update Matahari
    if (sunData.mc) {
        sunData.mc.scaleX = sunData.mc.scaleY = sunCurrentScale; 
        sunData.mc.x = centerX; 
        sunData.mc.y = centerY;
        sunData.depthZ = 0;
        depthList.push(sunData);
        sunData.mc.rotation += sunData.rot_deg_per_sec * dt;
        sunData.label.x = sunData.mc.x + 15;
        sunData.label.y = sunData.mc.y - 10;
        sunData.label.text = sunData.name + "\nOrbit: 0.00 px/s\nRotasi: " + sunData.rot_deg_per_sec.toFixed(4) + " deg/s";
        sunData.label.visible = !isZoomed;
    }

    // Update Planet
    for (var i:int=0; i<planets.length; i++) {
        var p:Object = planets[i];
        if (!p.mc) continue; 
        p.angle += p.omega * dt;

        // 3D Projection
        var localX:Number = p.a * Math.cos(p.angle);
        var localY:Number = p.b * Math.sin(p.angle);
        var localZ:Number = 0;
        var cosI:Number = Math.cos(p.inc);
        var sinI:Number = Math.sin(p.inc);
        var px:Number = localX;
        var py:Number = localY * cosI - localZ * sinI;
        var pz:Number = localY * sinI + localZ * cosI;
        var xCam:Number = px * cosA - pz * sinA;
        var zCam:Number = px * sinA + pz * cosA;
        var yCam:Number = py * cosT - zCam * sinT;
        zCam = py * sinT + zCam * cosT;
        p.xCam = xCam;
        p.yCam = yCam;

        var persp:Number = camZoom / (camZoom + zCam);
        var sx:Number = centerX + xCam * persp;
        var sy:Number = centerY + yCam * persp;

        p.mc.x = sx;
        p.mc.y = sy;
        p.mc.scaleX = p.mc.scaleY = p.baseScale * persp; 
        p.mc.rotation += p.rot_deg_per_sec * dt;
        
        var speed:Number = (2 * Math.PI * p.a) / p.periodSec;
        var displaySpeed:String = speed.toFixed(2);
        
        p.label.x = sx + 15;
        p.label.y = sy - 10;
        p.label.text = p.name + "\nOrbit: " + displaySpeed + " px/s\nRotasi: " + p.rot_deg_per_sec.toFixed(4) + " deg/s";
        p.label.visible = !isZoomed;

        p.depthZ = zCam;
        depthList.push(p);

      
        // Depth Fog/Kabut Jauh (Mengurangi Alpha jika planet terlalu jauh)
       
        var depthAlpha:Number = 1.0;
        if (p.depthZ > DEPTH_FOG_START) {
            var fogFactor:Number = (p.depthZ - DEPTH_FOG_START) / (DEPTH_FOG_END - DEPTH_FOG_START);
            fogFactor = Math.min(1.0, fogFactor);
            depthAlpha = 1.0 - (fogFactor * 0.5); // Maksimal kabur 50%
        }
        
        p.mc.alpha = depthAlpha;
        p.label.alpha = depthAlpha;

        // Alpha Orbit Dinamis (Fade Halus + Offset)
    
        var sinValue:Number = Math.sin(elapsedSec * orbitFadeSpeed + p.fadeOffset);
        var orbitDynamicAlpha:Number = orbitMinAlpha + (orbitMaxAlpha - orbitMinAlpha) * ((sinValue + 1) / 2);

        // Tambahkan efek kabut pada orbit
        var finalOrbitAlpha:Number = orbitDynamicAlpha * depthAlpha; 

        // orbit garis 3D menggunakan finalOrbitAlpha
        orbitLayer.graphics.lineStyle(2.5, p.color, finalOrbitAlpha);
        var steps:int = 100;
        for (var j:int=0; j<=steps; j++) {
            var ang:Number = j/steps*2*Math.PI;
            var lx:Number = p.a * Math.cos(ang);
            var ly:Number = p.b * Math.sin(ang);
            var lz:Number = 0;
            var ox:Number = lx;
            var oy:Number = ly * cosI - lz * sinI;
            var oz:Number = ly * sinI + lz * cosI;
            var xCam2:Number = ox * cosA - oz * sinA;
            var zCam2:Number = ox * sinA + oz * cosA;
            var yCam2:Number = oy * cosT - zCam2 * sinT;
            zCam2 = oy * sinT + zCam2 * cosT;
            var persp2:Number = camZoom / (camZoom + zCam2);
            var sx2:Number = centerX + xCam2 * persp2;
            var sy2:Number = centerY + yCam2 * persp2;
            if (j==0) orbitLayer.graphics.moveTo(sx2, sy2);
            else orbitLayer.graphics.lineTo(sx2, sy2);
        }
    }

    // Depth sorting
    depthList.sortOn("depthZ", Array.NUMERIC | Array.DESCENDING);
    for (var k:int=0; k<depthList.length; k++) {
        planetLayer.setChildIndex(depthList[k].mc, k);
    }
}
