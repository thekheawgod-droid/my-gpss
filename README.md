<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Wheel Mate - Rectangle Geofence Stable</title>
    
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.7.1/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.7.1/dist/leaflet.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>

    <style>
        #map { height: 100vh; width: 100%; background: #e9ecef; position: absolute; top: 0; left: 0; z-index: 1; }
        body { margin: 0; padding: 0; font-family: 'Kanit', sans-serif; overflow: hidden; }
        
        .ui-panel { 
            position: absolute; top: 10px; right: 10px; z-index: 1000; 
            background: rgba(255, 255, 255, 0.95); padding: 15px; border-radius: 12px; 
            box-shadow: 0 4px 15px rgba(0,0,0,0.3); width: 260px; max-height: 90vh; overflow-y: auto; 
        }
        button { width: 100%; padding: 10px; margin-top: 8px; cursor: pointer; border: none; border-radius: 6px; color: white; font-weight: bold; transition: 0.3s; }
        input, select { width: 100%; padding: 8px; margin-top: 5px; border: 1px solid #ddd; border-radius: 5px; box-sizing: border-box; }
        .btn-blue { background-color: #0d6efd; }
        .btn-green { background-color: #198754; }
        .btn-red { background-color: #dc3545; }
        .btn-dark { background-color: #212529; }
        .config-box { background: #f1f3f5; padding: 12px; border-radius: 8px; margin-top: 10px; border: 1px solid #dee2e6; }
        .status-text { font-size: 0.9em; margin-bottom: 5px; font-weight: bold; }
        .boundary-info { background: #fff3cd; color: #856404; padding: 10px; border-radius: 8px; margin-top: 10px; border: 1px solid #ffeeba; font-weight: bold; text-align: center; }
    </style>
</head>
<body>
    <div class="ui-panel">
        <div class="status-text">บอร์ด: <span id="status" style="color: gray;">กำลังเตรียมแผนที่...</span></div>
        <div id="boundaryDisplay" class="boundary-info">ห่างจากขอบ: - เมตร</div>
        <hr>
        
        <div class="config-box">
            <b>⚙️ ตั้งค่าขอบเขต (Geofence)</b><br>
            <label><small>รูปแบบพื้นที่:</small></label>
            <select id="selectShape">
                <option value="circle">วงกลม (Circle)</option>
                <option value="rectangle">สี่เหลี่ยม (Rectangle)</option>
            </select>
            
            <label><small>ระยะ/รัศมี (เมตร):</small></label>
            <input type="number" id="inputRadius" value="50">
            <p style="font-size: 0.8em; margin: 5px 0;">ศูนย์กลาง: <span id="displayCenter">ยังไม่ได้เลือก</span></p>
            <button class="btn-dark" id="btnToggleMode">🔓 เปิดโหมดจิ้มวางรั้ว</button>
            <button class="btn-green" id="btnSaveFence">💾 บันทึกขอบเขต</button>
        </div>

        <button class="btn-blue" id="btnShowHistory">🚩 แสดงเส้นทางย้อนหลัง</button>
        <button class="btn-red" id="btnDeleteDB">🗑️ ล้างฐานข้อมูล</button>
    </div>

    <div id="map"></div>

    <script>
        var map = L.map('map').setView([14.9874, 102.1180], 17);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
        
        var marker = L.marker([14.9874, 102.1180]).addTo(map);
        var polyline = L.polyline([], {color: 'red', weight: 4}).addTo(map);
        var fenceCircle = L.circle([0, 0], {radius: 0, color: 'blue', fillOpacity: 0.1}).addTo(map);
        var fenceRect = L.rectangle([[0,0],[0,0]], {color: 'blue', fillOpacity: 0.1}).addTo(map);

        const firebaseConfig = {
            apiKey: "AIzaSyA5PIqr8Ss64jWEDSAl0l7huc3yjYtM",
            databaseURL: "https://my-gps-tracker-2113a-default-rtdb.asia-southeast1.firebasedatabase.app"
        };
        firebase.initializeApp(firebaseConfig);
        const database = firebase.database();

        let currentFence = { type: "circle", lat: 14.9874, lng: 102.1180, radius: 50 };
        let isConfigMode = false;
        let lastHeartbeat = 0;

        // ดึงค่า Geofence ล่าสุดจาก Firebase
        database.ref('Settings/Geofence').on('value', (snapshot) => {
            const data = snapshot.val();
            if (data) {
                currentFence = data;
                document.getElementById('selectShape').value = data.type || "circle";
                updateFenceDisplay(data);
            }
        });

        function updateFenceDisplay(data) {
            fenceCircle.setStyle({opacity: 0, fillOpacity: 0});
            fenceRect.setStyle({opacity: 0, fillOpacity: 0});

            if (data.type === "rectangle") {
                // คำนวณขอบเขตสี่เหลี่ยมจากระยะทางเมตร
                let r = data.radius / 111320; 
                let bounds = [[data.lat - r, data.lng - r], [data.lat + r, data.lng + r]];
                fenceRect.setBounds(bounds);
                fenceRect.setStyle({opacity: 1, fillOpacity: 0.1, color: 'blue'});
            } else {
                fenceCircle.setLatLng([data.lat, data.lng]);
                fenceCircle.setRadius(data.radius);
                fenceCircle.setStyle({opacity: 1, fillOpacity: 0.1, color: 'blue'});
            }
            document.getElementById('displayCenter').innerText = data.lat.toFixed(4) + ", " + data.lng.toFixed(4);
        }

        document.getElementById('btnToggleMode').onclick = function() {
            isConfigMode = !isConfigMode;
            this.innerText = isConfigMode ? "🔒 ล็อคตำแหน่งรั้ว" : "🔓 เปิดโหมดจิ้มวางรั้ว";
            this.style.backgroundColor = isConfigMode ? "#ffc107" : "#212529";
        };

        map.on('click', function(e) {
            if (isConfigMode) {
                currentFence.lat = e.latlng.lat;
                currentFence.lng = e.latlng.lng;
                currentFence.type = document.getElementById('selectShape').value;
                currentFence.radius = parseInt(document.getElementById('inputRadius').value);
                updateFenceDisplay(currentFence);
            }
        });

        document.getElementById('btnSaveFence').onclick = function() {
            currentFence.type = document.getElementById('selectShape').value;
            currentFence.radius = parseInt(document.getElementById('inputRadius').value);
            
            // เพิ่มตรรกะการคำนวณ Bounds สำหรับบันทึกลง Firebase
            if (currentFence.type === "rectangle") {
                let r = currentFence.radius / 111320;
                currentFence.latMin = currentFence.lat - r;
                currentFence.latMax = currentFence.lat + r;
                currentFence.lngMin = currentFence.lng - r;
                currentFence.lngMax = currentFence.lng + r;
            }

            database.ref('Settings/Geofence').set(currentFence).then(() => {
                alert("บันทึกขอบเขตใหม่เรียบร้อย!");
                isConfigMode = false;
            });
        };

        database.ref('GPS').on('value', (snapshot) => {
            const data = snapshot.val();
            if (data && data.lat) {
                const pos = [data.lat, data.lng];
                marker.setLatLng(pos);
                if (!isConfigMode) map.panTo(pos);

                let isOutside = false;
                let distFromEdge = 0;

                // ตรรกะการตรวจจับตามรูปทรง
                if (currentFence.type === "rectangle") {
                    isOutside = (data.lat < currentFence.latMin || data.lat > currentFence.latMax || 
                                 data.lng < currentFence.lngMin || data.lng > currentFence.lngMax);
                    // สำหรับสี่เหลี่ยม ระยะพ้นขอบคำนวณจากด้านที่ใกล้ที่สุด (โดยประมาณ)
                    let dLat = Math.max(currentFence.latMin - data.lat, data.lat - currentFence.latMax, 0);
                    let dLng = Math.max(currentFence.lngMin - data.lng, data.lng - currentFence.lngMax, 0);
                    distFromEdge = Math.sqrt(dLat*dLat + dLng*dLng) * 111320;
                } else {
                    let distFromCenter = L.latLng(pos).distanceTo(L.latLng([currentFence.lat, currentFence.lng]));
                    distFromEdge = distFromCenter - currentFence.radius;
                    isOutside = (distFromEdge > 0);
                }

                let displayObj = document.getElementById('boundaryDisplay');
                if (isOutside) {
                    displayObj.innerHTML = `พ้นขอบออกมา: <span style="color:red">${distFromEdge.toFixed(1)} เมตร</span>`;
                    displayObj.style.background = "#f8d7da";
                    document.getElementById('status').innerText = "ALARM: นอกเขต!";
                    document.getElementById('status').style.color = "red";
                } else {
                    displayObj.innerHTML = `ห่างจากขอบ: <span style="color:green">${Math.abs(distFromEdge).toFixed(1)} เมตร</span>`;
                    displayObj.style.background = "#d4edda";
                    document.getElementById('status').innerText = "Online - ในเขต";
                    document.getElementById('status').style.color = "green";
                }
                lastHeartbeat = Date.now();
                document.getElementById('time').innerText = "อัปเดต: " + new Date().toLocaleTimeString();
            }
        });
        // ... (ปุ่มแสดงประวัติและลบฐานข้อมูลยังเหมือนเดิม)
    </script>
</body>
</html>
