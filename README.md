<br>

<h1 align="center" style="font-family: 'Montserrat', sans-serif; color: #007bff;">Auto GS Wave QR Code Generator for FreePBX</h1>

<p align="center">
A simple, self-hosted web tool to automatically generate Grandstream Wave provisioning QR codes directly from your FreePBX extensions.
</p>
<p align="center">
<img src="https://img.shields.io/badge/FreePBX-Compatible-orange" alt="FreePBX Compatible">
<img src="https://img.shields.io/badge/GS_Wave-Ready-brightgreen" alt="GS Wave Ready">
<img src="https://img.shields.io/badge/PHP-Required-blue" alt="PHP Required">
</p>

📋 Overview
This project provides a complete solution for administrators to quickly provision Grandstream Wave softphone clients. Instead of manually typing SIP credentials, this tool allows you to simply enter an extension number, fetch the credentials securely from your FreePBX database, and instantly generate a QR code that the GS Wave app can scan for one-touch configuration.

The system consists of two main components:

A PHP Backend: A secure script that runs on your FreePBX server to look up extension details.

An HTML Frontend: A user-friendly web page to enter the extension number and display the generated QR code.

✨ Features
One-Click Fetching: Automatically pulls UserID, AuthID, Password, and DisplayName from the FreePBX database.

Instant QR Code Generation: Creates a high-compatibility QR code recognized by the GS Wave app.

Secure: Uses a token-based authentication method to protect your PHP endpoint from public access.

Self-Hosted: No reliance on third-party services. Everything runs directly on your PBX.

Easy to Deploy: Simple setup with just two files.

🚀 Installation & Setup
Follow these steps carefully to get the generator up and running on your FreePBX server.

### Step 1: Find Your FreePBX Database Password
First, you'll need the password for the FreePBX database user. The PHP script needs this to read the extension data.

Log in to your FreePBX server via SSH.

Run the following command to display the database password:

Bash

grep "AMPDBPASS" /etc/freepbx.conf
The output will look something like \$amp_conf['AMPDBPASS'] = 'your_password_here';. Copy this password and save it securely for the next step.

### Step 2: Create the PHP Backend File
This script acts as the secure bridge to your database.

While in your SSH session, navigate to the web root directory:

Bash

cd /var/www/html/
Create a new file named get_extension_details.php using nano (or your preferred editor):

Bash

nano get_extension_details.php
Copy the entire code block below and paste it into the nano editor.

IMPORTANT: You must edit two placeholders in this file:

YourSecretTokenHere12345! → Change this to a long, random string. This is your private key.

YOUR_FREEPBX_DB_PASSWORD → Replace this with the password you retrieved in Step 1.

PHP

<?php
// File: /var/www/html/get_extension_details.php (FINAL VERSION)

// --- CONFIGURATION & SECURITY ---
$required_token = 'YourSecretTokenHere12345!'; // <-- CHANGE THIS TO A STRONG, RANDOM KEY

// Set headers
header('Content-Type: application/json');

// 1. Check for the security token
if (!isset($_GET['token']) || $_GET['token'] !== $required_token) {
    http_response_code(403);
    echo json_encode(['error' => 'Access Denied: Invalid security token.']);
    exit();
}

// 2. Get the extension number
if (!isset($_GET['ext'])) {
    http_response_code(400);
    echo json_encode(['error' => 'Extension number not provided.']);
    exit();
}
$extension_number = $_GET['ext'];

// 3. Connect to the FreePBX Database
$db_host = 'localhost';
$db_name = 'asterisk';
$db_user = 'freepbxuser';
$db_pass = 'YOUR_FREEPBX_DB_PASSWORD'; // <-- PASTE THE PASSWORD FROM STEP 1 HERE

try {
    $pdo = new PDO("mysql:host=$db_host;dbname=$db_name", $db_user, $db_pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch (PDOException $e) {
    http_response_code(500);
    echo json_encode(['error' => 'Database connection failed.']);
    exit();
}

// 4. Query the database for older chan_sip extensions
$stmt = $pdo->prepare("
    SELECT
        u.extension AS userID,
        s.data AS authPass,
        u.name AS displayName
    FROM users u
    LEFT JOIN sip s ON u.extension = s.id
    WHERE u.extension = ? AND s.keyword = 'secret'
");

$stmt->execute([$extension_number]);
$result = $stmt->fetch(PDO::FETCH_ASSOC);

// 5. Return the result as JSON
if ($result) {
    $registerServer = $_SERVER['SERVER_NAME'] . ":5060";
    $data = [
        'registerServer' => $registerServer,
        'userID' => $result['userID'],
        'authID' => $result['userID'],
        'authPass' => $result['authPass'],
        'accountName' => $result['userID'],
        'displayName' => $result['displayName'],
        'voicemail' => '*97'
    ];
    echo json_encode($data);
} else {
    http_response_code(404);
    echo json_encode(['error' => 'Extension not found.']);
}
?>
Save the file and exit nano by pressing CTRL + X, then Y, then Enter.

### Step 3: Create the HTML Frontend File
This is the web page you will use to generate the codes.

In the same directory (/var/www/html/), create the HTML file. You can name it qr.html or index.html.

Bash

nano qr.html
Copy the entire code block below and paste it into the editor.

IMPORTANT: Make sure the value of const apiToken in this file exactly matches the $required_token you set in the PHP file.

HTML

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GS Wave QR Generator for FreePBX</title>
    <link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@400;700&family=Fira+Mono&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Montserrat', Arial, sans-serif; margin: 0; padding: 20px; background: #f5f8fa; color: #333; display: flex; flex-direction: column; align-items: center; }
        .container { width: 100%; max-width: 600px; }
        h1 { color: #007bff; font-weight: 700; font-size: 2.4em; text-align: center; margin-bottom: 25px; }
        .input-group, .fetch-group { margin-bottom: 16px; }
        label { display: block; margin-bottom: 6px; font-weight: 700; color: #555; }
        input[type="text"], input[type="number"] { width: 100%; box-sizing: border-box; padding: 10px; border: 1px solid #bbb; border-radius: 6px; font-size: 1em; }
        input:read-only { background-color: #e9ecef; }
        button { padding: 12px 20px; background-color: #007bff; color: white; border: none; border-radius: 6px; cursor: pointer; font-size: 1.1em; font-weight: 700; display: block; width: 100%; margin-top: 20px; }
        button:hover { background-color: #0056b3; }
        .fetch-group { display: flex; gap: 10px; align-items: flex-end; }
        .fetch-group div { flex-grow: 1; }
        .fetch-group button { margin-top: 0; background-color: #28a745; }
        .fetch-group button:hover { background-color: #218838; }
        .status-message { margin-top: 10px; font-weight: bold; min-height: 20px; }
        .error { color: #dc3545; }
        .success { color: #28a745; }
        #qrcode-container { margin-top: 30px; border: 1px solid #e3e6ea; padding: 20px; display: none; flex-direction: column; align-items: center; background: white; border-radius: 10px; }
        #qrcode-container h2, #qrcode-container h3 { color: #007bff; margin-top: 0; margin-bottom: 12px; font-size: 1.3em; text-align: center; }
        #qrcode-text { white-space: pre-wrap; word-break: break-all; background-color: #eaf1fb; padding: 14px; border-radius: 4px; margin-top: 16px; font-family: 'Fira Mono', monospace; font-size: 1em; max-width: 100%; color: #234; }
    </style>
</head>
<body>
    <div class="container">
        <h1>GS Wave QR Generator for FreePBX</h1>
        <div class="fetch-group">
            <div>
                <label for="fetchExtension">Enter Extension Number:</label>
                <input type="number" id="fetchExtension" placeholder="e.g., 1004">
            </div>
            <button onclick="fetchExtensionDetails()">Fetch Details</button>
        </div>
        <div id="status" class="status-message"></div>
        <hr style="margin: 25px 0;">
        <div class="input-group"><label for="registerServer">RegisterServer:</label><input type="text" id="registerServer" readonly></div>
        <div class="input-group"><label for="userID">UserID:</label><input type="text" id="userID" readonly></div>
        <div class="input-group"><label for="authID">AuthID:</label><input type="text" id="authID" readonly></div>
        <div class="input-group"><label for="authPass">AuthPass:</label><input type="text" id="authPass" readonly></div>
        <div class="input-group"><label for="accountName">AccountName:</label><input type="text" id="accountName" readonly></div>
        <div class="input-group"><label for="displayName">DisplayName:</label><input type="text" id="displayName" readonly></div>
        <div class="input-group"><label for="voicemail">Voicemail:</label><input type="text" id="voicemail" readonly></div>
        <div id="qrcode-container">
            <h2>Generated QR Code:</h2>
            <canvas id="qrCanvas"></canvas>
            <h3>QR Code Content (XML):</h3>
            <pre id="qrcode-text"></pre>
        </div>
    </div>
    <script src="https://unpkg.com/qrious@4.0.2/dist/qrious.js"></script>
    <script>
        const apiUrl = 'get_extension_details.php';
        const apiToken = 'YourSecretTokenHere12345!'; // <-- MUST MATCH the token in your PHP file

        const statusDiv = document.getElementById('status');
        const qrContainer = document.getElementById('qrcode-container');

        async function fetchExtensionDetails() {
            const ext = document.getElementById('fetchExtension').value;
            statusDiv.textContent = '';
            qrContainer.style.display = 'none';

            if (!ext) {
                statusDiv.className = 'status-message error';
                statusDiv.textContent = 'Please enter an extension number.';
                return;
            }

            statusDiv.className = 'status-message';
            statusDiv.textContent = 'Fetching details...';

            try {
                const fullUrl = `${apiUrl}?ext=${ext}&token=${apiToken}`;
                const response = await fetch(fullUrl);
                const data = await response.json();

                if (!response.ok) {
                    throw new Error(data.error || `Error ${response.status}`);
                }

                document.getElementById('registerServer').value = data.registerServer || '';
                document.getElementById('userID').value = data.userID || '';
                document.getElementById('authID').value = data.authID || '';
                document.getElementById('authPass').value = data.authPass || '';
                document.getElementById('accountName').value = data.accountName || '';
                document.getElementById('displayName').value = data.displayName || '';
                document.getElementById('voicemail').value = data.voicemail || '';

                statusDiv.className = 'status-message success';
                statusDiv.textContent = `Successfully fetched details for extension ${ext}.`;
                generateQrCode();

            } catch (error) {
                statusDiv.className = 'status-message error';
                statusDiv.textContent = `Failed to fetch details: ${error.message}`;
            }
        }

        function generateQrCode() {
            const registerServer = document.getElementById('registerServer').value;
            if (!registerServer) {
                qrContainer.style.display = 'none';
                return;
            }

            const xmlContent = `<?xml version="1.0" encoding="utf-8"?><AccountConfig version="1"><Account><RegisterServer>${registerServer}</RegisterServer><UserID>${document.getElementById('userID').value}</UserID><AuthID>${document.getElementById('authID').value}</AuthID><AuthPass>${document.getElementById('authPass').value}</AuthPass><AccountName>${document.getElementById('accountName').value}</AccountName><DisplayName>${document.getElementById('displayName').value}</DisplayName><Voicemail>${document.getElementById('voicemail').value}</Voicemail></Account></AccountConfig>`;

            document.getElementById('qrcode-text').textContent = xmlContent;
            qrContainer.style.display = 'flex';

            new QRious({
                element: document.getElementById('qrCanvas'),
                value: xmlContent,
                size: 250,
                padding: 10,
                level: 'H'
            });
        }
    </script>
</body>
</html>
Save the file and exit nano (CTRL + X, Y, Enter).

💻 How to Use
Your QR Code Generator is now live!

Open your web browser and navigate to the IP address of your FreePBX server, followed by the name of your HTML file. For example:

http://192.168.1.100/qr.html

Enter a valid extension number in the input box.

Click the "Fetch Details" button.

The tool will securely connect to the database, populate the fields, and instantly display the QR code.

Open the Grandstream Wave app on your mobile device, choose "UCM Account (Scan QR Code)," and scan the code on your screen.

Your extension is now provisioned! 🎉

Troubleshooting & Notes
PJSIP Extensions: The provided PHP script is configured for older chan_sip extensions, which is common. If you use PJSIP and the tool reports "Extension not found," you will need to adjust the SQL query in get_extension_details.php.

Security: It is highly recommended to restrict access to these files, for example, by using .htaccess rules to only allow access from your local network.

Permissions: If you encounter a "Forbidden" error when trying to access the pages, you may need to adjust file permissions. Ensure the web server user (usually asterisk or apache) can read the files:

Bash

chown asterisk:asterisk /var/www/html/get_extension_details.php
chown asterisk:asterisk /var/www/html/qr.html
<br>
