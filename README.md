📂 Project Folder StructureCreate this file structure locally before deploying:textticket-system/
├── api/
│   ├── book.js
│   └── verify.js
├── public/
│   ├── index.html
│   └── scanner.html
└── package.json
Use code with caution.📦 1. Configuration (package.json)This handles dependencies. Vercel automatically installs these when deploying [1].json{
  "name": "vercel-qr-ticket-system",
  "version": "1.0.0",
  "dependencies": {
    "mongoose": "^8.0.0",
    "qrcode": "^1.5.3"
  }
}
Use code with caution.🗄️ 2. The Database Connection HelperBecause serverless functions wake up and shut down constantly, we must reuse existing MongoDB connection pools to avoid crashing your database. Create a folder named utils and place this file inside it.File: utils/db.jsjavascriptconst mongoose = require('mongoose');

let cachedConnection = null;

async function connectDB() {
    if (cachedConnection) return cachedConnection;

    if (!process.env.MONGODB_URI) {
        throw new Error('Please add your MONGODB_URI to environment variables');
    }

    cachedConnection = await mongoose.connect(process.env.MONGODB_URI);
    return cachedConnection;
}

// Define Schema here globally to avoid compilation errors on hot reloads
const TicketSchema = new mongoose.Schema({
    eventId: String,
    passengerName: String,
    passengerEmail: String,
    bookingDate: { type: Date, default: Date.now },
    isScanned: { type: Boolean, default: false },
    scannedAt: Date
});

const Ticket = mongoose.models.Ticket || mongoose.model('Ticket', TicketSchema);

module.exports = { connectDB, Ticket };
Use code with caution.⚙️ 3. Backend Functions (api/ folder)File 1: api/book.js (Handles Ticket Generation)javascriptconst { connectDB, Ticket } = require('../utils/db');
const QRCode = require('qrcode');

// Helper to manage CORS headers in Vercel
function setCors(res) {
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type');
}

module.exports = async (req, res) => {
    setCors(res);
    if (req.method === 'OPTIONS') return res.status(200).end();
    if (req.method !== 'POST') return res.status(405).json({ message: 'Method Not Allowed' });

    try {
        await connectDB();
        const { eventId, name, email } = req.body;

        const newTicket = new Ticket({ eventId, passengerName: name, passengerEmail: email });
        await newTicket.save();

        // Dynamically detects the current deployed Vercel domain URL
        const protocol = req.headers['x-forwarded-proto'] || 'http';
        const host = req.headers['x-forwarded-host'] || req.headers.host;
        const validationUrl = `${protocol}://${host}/api/verify?id=${newTicket._id}`;
        
        const qrCodeDataUrl = await QRCode.toDataURL(validationUrl);

        return res.status(201).json({
            success: true,
            ticketId: newTicket._id,
            qrCode: qrCodeDataUrl
        });
    } catch (error) {
        return res.status(500).json({ success: false, message: error.message });
    }
};
Use code with caution.File 2: api/verify.js (Handles Scanning & Ticket Check-in)javascriptconst { connectDB, Ticket } = require('../utils/db');

function setCors(res) {
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'GET, OPTIONS');
}

module.exports = async (req, res) => {
    setCors(res);
    if (req.method === 'OPTIONS') return res.status(200).end();
    if (req.method !== 'GET') return res.status(405).send('Method Not Allowed');

    try {
        await connectDB();
        const { id } = req.query;

        if (!id) return res.status(400).send('<h1>❌ Missing Ticket ID</h1>');

        const ticket = await Ticket.findById(id);

        if (!ticket) {
            return res.status(404).send('<h1>❌ Invalid Ticket</h1><p>This ticket does not exist in the system.</p>');
        }

        if (ticket.isScanned) {
            return res.status(400).send(`
                <div style="text-align:center; font-family:sans-serif; margin-top:50px;">
                    <h1 style="color:orange;">⚠️ Already Used</h1>
                    <p>Ticket Holder: <strong>${ticket.passengerName}</strong></p>
                    <p>Scanned on: ${new Date(ticket.scannedAt).toLocaleString()}</p>
                </div>
            `);
        }

        ticket.isScanned = true;
        ticket.scannedAt = new Date();
        await ticket.save();

        return res.status(200).send(`
            <div style="text-align:center; font-family:sans-serif; margin-top:50px;">
                <h1 style="color:green;">✅ Ticket Valid</h1>
                <p>Welcome, <strong>${ticket.passengerName}</strong>!</p>
                <p>Event ID: ${ticket.eventId}</p>
            </div>
        `);
    } catch (error) {
        return res.status(500).send('<h1>❌ Verification Server Error</h1>');
    }
};
Use code with caution.🎨 4. Frontend Client (public/ folder)Vercel automatically serves any static HTML files located inside the public directory [2].File 1: public/index.html (The Booking Form)html<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Event Booking Portal</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 400px; margin: 50px auto; padding: 20px; text-align: center; }
        input { width: 100%; padding: 10px; margin: 10px 0; box-sizing: border-box; }
        button { width: 100%; padding: 10px; background: #0070f3; color: white; border: none; cursor: pointer; font-size: 16px; }
        #ticketResult { display:none; margin-top:30px; border: 1px dashed #ccc; padding: 20px; }
        #qrImage { width: 200px; height: 200px; }
    </style>
</head>
<body>
    <h2>🎟️ Event Registration</h2>
    <form id="bookingForm">
        <input type="text" id="name" placeholder="Your Full Name" required>
        <input type="email" id="email" placeholder="Your Email Address" required>
        <button type="submit">Generate E-Ticket</button>
    </form>

    <div id="ticketResult">
        <h3>Your Entry Pass</h3>
        <img id="qrImage" src="" alt="QR Entry Code">
        <p>Screenshot this code and present it at the entry gate.</p>
    </div>

    <script>
        document.getElementById('bookingForm').addEventListener('submit', async (e) => {
            e.preventDefault();
            const name = document.getElementById('name').value;
            const email = document.getElementById('email').value;

            // Uses relative URLs because API routes live on the same Vercel deployment
            const response = await fetch('/api/book', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ eventId: "FESTIVAL-2026", name, email })
            });

            const data = await response.json();
            if(data.success) {
                document.getElementById('qrImage').src = data.qrCode;
                document.getElementById('ticketResult').style.display = 'block';
                document.getElementById('bookingForm').reset();
            } else {
                alert("Booking failed: " + data.message);
            }
        });
    </script>
</body>
</html>
Use code with caution.File 2: public/scanner.html (The Gate Scanner App)html<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gate Control Scanner</title>
    <script src="https://unpkg.com"></script>
    <style>
        body { font-family: Arial, sans-serif; display: flex; flex-direction: column; align-items: center; margin-top: 40px; }
        #reader { width: 100%; max-width: 450px; background: black; }
        #result-frame { width: 100%; max-width: 450px; margin-top: 20px; }
        .btn { padding: 10px 20px; background: #333; color: white; border: none; cursor: pointer; margin-top: 15px; display: none; }
    </style>
</head>
<body>
    <h2>📷 Gate Access Scanner</h2>
    <div id="reader"></div>
    <div id="result-frame"></div>
    <button id="resetBtn" class="btn" onclick="location.reload()">Scan Next Ticket</button>

    <script>
        function onScanSuccess(decodedText) {
            html5QrcodeScanner.clear(); // Freeze camera stream instantly on successful read
            document.getElementById('resetBtn').style.display = 'block';
            
            const resultFrame = document.getElementById('result-frame');
            resultFrame.innerHTML = "<h3>Verifying with Cloud Database...</h3>";
            
            // Call the Vercel serverless verification endpoint decoded from the QR code
            fetch(decodedText)
                .then(res => res.text())
                .then(htmlResponse => {
                    resultFrame.innerHTML = htmlResponse; 
                })
                .catch(err => {
                    resultFrame.innerHTML = "<h3 style='color:red;'>Network error verifying ticket.</h3>";
                });
        }

        let html5QrcodeScanner = new Html5QrcodeScanner("reader", { fps: 15, qrbox: 250 });
        html5QrcodeScanner.render(onScanSuccess);
    </script>
</body>
</html>
Use code with caution.
