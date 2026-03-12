<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GD Auto-Bot Generator V11</title>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #1a1a2e; color: #ffffff; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
        .container { background-color: #16213e; padding: 30px; border-radius: 10px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.5); text-align: center; width: 450px; }
        .bot-panel { background-color: #0f3460; padding: 15px; border-radius: 8px; margin-top: 20px; border: 2px dashed #e94560; }
        input { width: 90%; padding: 10px; margin: 10px 0; border: none; border-radius: 5px; font-size: 16px; }
        button { background-color: #e94560; color: white; border: none; padding: 12px 20px; margin-top: 10px; border-radius: 5px; font-size: 16px; cursor: pointer; transition: 0.3s; width: 100%; font-weight: bold; }
        button:hover { background-color: #ff5c77; }
        button.active { background-color: #4caf50; }
        .log-box { margin-top: 15px; background-color: #1a1a2e; padding: 10px; border-radius: 5px; font-size: 13px; color: #a8b2d1; text-align: left; height: 60px; overflow-y: auto; }
        #status { font-weight: bold; color: #ff5c77; margin-top: 10px; }
    </style>
</head>
<body>

<div class="container">
    <h2>GD Level Generator V11</h2>
    <p>Maximum Chaos Edition (Still Playable!)</p>
    
    <input type="number" id="levelChunks" placeholder="Number of chunks (Length)" value="60">
    <button onclick="generateAndDownload(false)">Generate One Now</button>
    
    <div class="bot-panel">
        <h3>🤖 Auto-Bot Mode</h3>
        <input type="number" id="timerMinutes" placeholder="Minutes between uploads" value="30">
        <button id="botBtn" onclick="toggleBot()">Start Auto-Bot</button>
        <div id="status">Bot is OFF</div>
        
        <div class="log-box" id="botLog"><em>Waiting for bot to generate...</em></div>
    </div>
</div>

<script>
    // --- MASSIVE OBJECT POOLS ---
    const POOLS = { 
        BLOCKS: [1, 2, 3, 4, 5, 6, 7, 154, 155, 156, 157, 158, 250, 251, 252, 253, 254], 
        SPIKES: [8, 9, 39, 103, 151, 258, 259], 
        DECO: [44, 211, 152, 501, 502, 503, 504, 505, 19, 20, 21, 22],
        YELLOW_PAD: [35], YELLOW_ORB: [36], PINK_ORB: [141],
        SHIP_PORTAL: [12], CUBE_PORTAL: [13], GRAVITY_DOWN: [10], GRAVITY_UP: [11],
        SPEEDS: [200, 201, 202] 
    };

    const adjectives = ["Glitch", "Corrupt", "Cursed", "Fractal", "Broken", "Infinite", "Random", "Absurd", "Viral", "Chaotic"];
    const nouns = ["Algorithm", "Dream", "Error", "Matrix", "Loop", "Syntax", "Entity", "Bytes", "Void", "Engine"];

    let botInterval = null;

    // --- HELPER FUNCTIONS ---
    // Added Property 6 for Rotation!
    function createObject(id, x, y, scale = 1, zOrder = 1, rotation = 0) { 
        return `1,${id},2,${x},3,${y},32,${scale},25,${zOrder},6,${rotation};`; 
    }
    
    function createColorTrigger(x, y, r, g, b, duration) {
        return `1,29,2,${x},3,${y},23,1,7,${r},8,${g},9,${b},10,${duration};`;
    }

    function getRand(arr) { return arr[Math.floor(Math.random() * arr.length)]; }
    function getRandomName() { return `${getRand(adjectives)} ${getRand(nouns)}`; }
    function logAction(msg) { const logBox = document.getElementById("botLog"); logBox.innerHTML = `<div>> ${msg}</div>` + logBox.innerHTML; }

    function scatterDeco(startX, length) {
        let str = "";
        if (Math.random() > 0.2) { // 80% chance to scatter deco
            let numDeco = Math.floor(Math.random() * 8) + 3; // 3 to 10 decorations
            for(let i=0; i<numDeco; i++) {
                let decoID = getRand(POOLS.DECO);
                let x = startX + Math.floor(Math.random() * length);
                let y = Math.floor(Math.random() * 300) + 15; // Float anywhere on screen
                let scale = (Math.random() * 2.0 + 0.5).toFixed(2); 
                let rot = Math.floor(Math.random() * 360); // Crazy angles!
                str += createObject(decoID, x, y, scale, -1, rot); 
            }
        }
        return str;
    }

    // --- CHAOTIC CHUNKS ---
    function chunkFlat(x, y) {
        let len = Math.floor(Math.random() * 5) + 3; // Random length between 3 and 7 blocks
        let str = scatterDeco(x, len * 30);
        for(let i=0; i<len; i++) {
            // Pick a completely random block texture for EVERY single space
            str += createObject(getRand(POOLS.BLOCKS), x + (i*30), y);
            // Fill underneath so floating pillars look solid
            for(let fillY = y-30; fillY >= 15; fillY -= 30) {
                str += createObject(getRand(POOLS.BLOCKS), x + (i*30), fillY);
            }
        }
        return { data: str, nextX: x + (len * 30), nextY: y };
    }
    
    function chunkSingleSpike(x, y) {
        let str = scatterDeco(x, 150);
        for(let i=0; i<5; i++) str += createObject(getRand(POOLS.BLOCKS), x + (i*30), y);
        str += createObject(getRand(POOLS.SPIKES), x + 60, y + 30); 
        return { data: str, nextX: x + 150, nextY: y };
    }
    
    function chunkPadJump(x, y) {
        let str = scatterDeco(x, 330);
        str += createObject(getRand(POOLS.BLOCKS), x, y);
        str += createObject(getRand(POOLS.YELLOW_PAD), x, y + 30); 
        for(let i=4; i<=10; i++) str += createObject(getRand(POOLS.BLOCKS), x + (i*30), y);
        return { data: str, nextX: x + 330, nextY: y };
    }

    function chunkOrbJump(x, y) {
        let str = scatterDeco(x, 240);
        str += createObject(getRand(POOLS.BLOCKS), x, y);
        // Place a random Orb (Yellow or Pink) in the air
        let orbID = Math.random() > 0.5 ? getRand(POOLS.YELLOW_ORB) : getRand(POOLS.PINK_ORB);
        str += createObject(orbID, x + 60, y + 60); 
        // Catch the player on the other side of the gap
        for(let i=4; i<=9; i++) str += createObject(getRand(POOLS.BLOCKS), x + (i*30), y);
        return { data: str, nextX: x + 270, nextY: y };
    }
    
    function chunkSpeedChange(x, y) {
        let str = chunkFlat(x, y).data; // Just layer a speed change over a flat section
        str += createObject(getRand(POOLS.SPEEDS), x + 30, y + 30); 
        return { data: str, nextX: x + 150, nextY: y }; // Assumed 5 blocks wide
    }

    function chunkStairsUp(x, y) {
        let str = scatterDeco(x, 150);
        let heightChange = Math.random() > 0.5 ? 30 : 60; // Step up 1 or 2 blocks
        str += createObject(getRand(POOLS.BLOCKS), x, y);
        str += createObject(getRand(POOLS.BLOCKS), x + 60, y + heightChange);
        for(let i=3; i<=6; i++) {
            str += createObject(getRand(POOLS.BLOCKS), x + (i*30), y + heightChange);
            // Fill underneath
            for(let fillY = y + heightChange - 30; fillY >= 15; fillY -= 30) {
                str += createObject(getRand(POOLS.BLOCKS), x + (i*30), fillY);
            }
        }
        return { data: str, nextX: x + 210, nextY: y + heightChange };
    }

    function chunkStairsDown(x, y) {
        if (y <= 15) return chunkFlat(x, y); // Can't go lower than ground
        let str = scatterDeco(x, 150);
        let heightChange = 30; // Step down
        str += createObject(getRand(POOLS.BLOCKS), x, y);
        str += createObject(getRand(POOLS.BLOCKS), x + 60, y - heightChange);
        for(let i=3; i<=6; i++) {
            str += createObject(getRand(POOLS.BLOCKS), x + (i*30), y - heightChange);
            for(let fillY = y - heightChange - 30; fillY >= 15; fillY -= 30) {
                str += createObject(getRand(POOLS.BLOCKS), x + (i*30), fillY);
            }
        }
        return { data: str, nextX: x + 210, nextY: y - heightChange };
    }

    const chunkLibrary = [
        chunkFlat, chunkFlat, chunkSingleSpike, chunkPadJump, 
        chunkOrbJump, chunkSpeedChange, chunkStairsUp, chunkStairsDown
    ];

    // --- LEVEL GENERATOR ---
    function generateLevelString(numChunks) {
        let levelString = createColorTrigger(0, 150, Math.floor(Math.random()*255), Math.floor(Math.random()*255), Math.floor(Math.random()*255), 0.5);
        let currentX = 300;       
        let currentY = 45; // Start slightly elevated for maximum stair action
        
        for (let i = 0; i < numChunks; i++) {
            // Aggressive Color Changes
            if (Math.random() < 0.35) {
                let r = Math.floor(Math.random() * 256), g = Math.floor(Math.random() * 256), b = Math.floor(Math.random() * 256);
                levelString += createColorTrigger(currentX, currentY + 150, r, g, b, (Math.random() * 0.5).toFixed(2));
            }

            let result = getRand(chunkLibrary)(currentX, currentY);
            levelString += result.data;
            currentX = result.nextX;
            currentY = result.nextY;

            // Clamp Y so it doesn't go off into outer space or underground
            if (currentY > 250) currentY = 250;
            if (currentY < 15) currentY = 15;
        }
        return levelString;
    }

    function createGMDData(name, desc, levelString, songID) {
        return `<?xml version="1.0"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>kCEK</key><integer>4</integer>
    <key>k2</key><string>${name}</string>
    <key>k3</key><string>${desc}</string>
    <key>k4</key><string>${levelString}</string>
    <key>k8</key><integer>${songID}</integer> 
</dict>
</plist>`;
    }

    function generateAndDownload(isBot) {
        const chunks = parseInt(document.getElementById("levelChunks").value) || 60;
        const randomSongID = Math.floor(Math.random() * 22);
        const genName = getRandomName();

        const rawLevelString = generateLevelString(chunks);
        const gmdContent = createGMDData(genName, "Absolute AI Chaos.", rawLevelString, randomSongID);

        const blob = new Blob([gmdContent], { type: "text/xml" });
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = genName.replace(/[^a-z0-9]/gi, '_').toLowerCase() + ".gmd";
        document.body.appendChild(a);
        a.click();
        
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
        logAction(`Generated: <b>${genName}</b>`);
    }

    function toggleBot() {
        const btn = document.getElementById("botBtn");
        const statusText = document.getElementById("status");
        const minutes = parseFloat(document.getElementById("timerMinutes").value) || 30;

        if (botInterval) {
            clearInterval(botInterval);
            botInterval = null;
            btn.innerText = "Start Auto-Bot";
            btn.classList.remove("active");
            statusText.innerText = "Bot is OFF";
            statusText.style.color = "#ff5c77";
            logAction("Bot manually stopped.");
        } else {
            generateAndDownload(true); 
            botInterval = setInterval(() => { generateAndDownload(true); }, minutes * 60 * 1000);
            btn.innerText = "Stop Auto-Bot";
            btn.classList.add("active");
            statusText.innerText = `Bot is ON (Running every ${minutes} min)`;
            statusText.style.color = "#4caf50";
        }
    }
</script>

</body>
</html>
