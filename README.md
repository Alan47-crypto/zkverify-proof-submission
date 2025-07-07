# 🔐 zkVerify Automated Proof Submission Guide (Groth16)

> ✅ A complete, working guide to automatically generate and submit thousands of unique ZK proofs to the [zkVerify Incentivized Testnet](https://points.zkverify.io/loyalty?referral_code=KUO7D211) using Circom, SnarkJS, and a powerful automation script.

This guide provides a full end-to-end workflow, from installing the necessary tools to running a background script that handles everything for you.

---

## 📋 Prerequisites

Before you begin, make sure you have the following installed on your server (e.g., a Linux VPS):

* **Node.js**: Version 18+ is required.
* **API Key**: You'll need an API key from zkVerify. See Step 5 for instructions.

---

## 🚀 End-to-End Setup and Automation

Follow these steps to set up the project and start the automation script.

### Step 1: Install Global Tools

First, install `circom` and `snarkjs` globally using `npm`.

```bash
npm install -g circom
npm install -g snarkjs
```

### Step 2: Set Up Project Folders
Create the directory structure for the project.

```bash
# Create the main project folder and enter it
mkdir zkverify-automator && cd zkverify-automator

# Create sub-folders for proofs and data
mkdir real-proof data
```

### Step 3: Create the ZK Circuit
Next, create the simple sum.circom circuit file. This circuit will add two numbers, a and b.

```bash
cat > real-proof/sum.circom <<'EOF'
template SumCircuit() {
    signal input a;
    signal input b;
    signal output c;

    c <== a + b;
}

component main = SumCircuit();
EOF
```

### Step 4: One-Time Key Generation Ceremony
This multi-part step compiles the circuit and performs the trusted setup to generate the necessary proving and verification keys. This only needs to be done once.

```bash
# Navigate into the proof directory
cd real-proof

# 1. Compile the circuit
circom sum.circom --r1cs --wasm --sym -o .

# 2. Start the Powers of Tau ceremony
snarkjs powersoftau new bn128 12 pot12_0000.ptau -v

# 3. Contribute randomness (entropy)
# Note: You will be prompted to enter random text. Type anything and press Enter.
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="My Contribution" -v

# 4. Prepare for the final phase
snarkjs powersoftau prepare phase2 pot12_0001.ptau pot12_final.ptau -v

# 5. Generate the final proving key (.zkey)
snarkjs groth16 setup sum.r1cs pot12_final.ptau sum.zkey

# 6. Export the verification key to JSON format
snarkjs zkey export verificationkey sum.zkey verification_key.json
```

### Step 5: Prepare API Submission Scripts
Now, create the Node.js script and its configuration files.

```bash
# Navigate back to the main project directory
cd ..

# 1. Initialize the Node.js project
npm init -y && npm pkg set type=module

# 2. Install dependencies
npm install axios dotenv
```

#### Set Your API Key
Important: To have your submissions tracked for the incentivized testnet and earn points, you must use your own unique API key.

➡️ Request your key here: zkVerify Discord

Once you have your key, create the environment file by running the two commands below.

Create the .env file with a placeholder:

```bash
echo "API_KEY=YOUR_API_KEY_HERE" > .env
```

Edit the file and paste your key:

```bash
nano .env
```

Replace YOUR_API_KEY_HERE with the actual key you received, then save and exit (Ctrl+X, then Y, then Enter).

🧪 For Quick Testing (Optional)

If you just want to run a quick test without your submissions being tracked for points, you can use the default public API key with this single command:

```bash
echo "API_KEY=598f259f5f5d7476622ae52677395932fa98901f" > .env
```

#### Create the Node.js Submission Script (index.js)
This command creates the script file that sends your proof to the relayer.

```bash
cat << 'EOF' > index.js
import axios from 'axios';
import fs from 'fs';
import dotenv from 'dotenv';
dotenv.config();

const API_URL = '[https://relayer-api.horizenlabs.io/api/v1](https://relayer-api.horizenlabs.io/api/v1)';

const proof = JSON.parse(fs.readFileSync("./data/proof.json"));
const publicInputs = JSON.parse(fs.readFileSync("./data/public.json"));
const key = JSON.parse(fs.readFileSync("./data/verification_key.json"));

async function main() {
    const params = {
        proofType: "groth16",
        vkRegistered: false,
        proofOptions: {
            library: "snarkjs",
            curve: "bn128"
        },
        proofData: {
            proof,
            publicSignals: publicInputs,
            vk: key
        }
    };

    try {
        const response = await axios.post(`${API_URL}/submit-proof/${process.env.API_KEY}`, params);
        console.log(JSON.stringify(response.data));

        const jobId = response.data.jobId;
        if (!jobId) return;

        while (true) {
            const status = await axios.get(`${API_URL}/job-status/${process.env.API_KEY}/${jobId}`);
            if (status.data.status === "Finalized" || status.data.status === "Failed") {
                console.log(JSON.stringify(status.data));
                break;
            }
            await new Promise(res => setTimeout(res, 5000));
        }
    } catch (err) {
        console.error(JSON.stringify(err.response?.data || { error: err.message }));
    }
}

main();
EOF
```

### Step 6: Create the Final Automation Script
This script ties everything together, running the process in a loop to submit many unique proofs. 

Check for jq:

```bash
# Check version
jq --version

# Install on Debian/Ubuntu if needed
sudo apt-get update && sudo apt-get install -y jq
```

Create automate.sh:

```bash
cat << 'EOF' > automate.sh
#!/bin/bash

# --- CONFIGURATION ---
TOTAL_PROOFS=1000
LOG_FILE="automation.log"
CSV_FILE="submissions.csv"

# --- SCRIPT START ---

echo "Starting zkVerify Automation Script (curl version)..."
if [ ! -f "$CSV_FILE" ]; then
    echo "Timestamp,Proof_Number,JobID,TxHash" > $CSV_FILE
fi
API_KEY=$(grep API_KEY .env | cut -d '=' -f2)
API_URL="[https://relayer-api.horizenlabs.io/api/v1](https://relayer-api.horizenlabs.io/api/v1)"

if [ -f "real-proof/verification_key.json" ]; then
    mv real-proof/verification_key.json data/
fi

for i in $(seq 1 $TOTAL_PROOFS)
do
    echo "--------------------------------------------------"
    echo "➡️ Starting Proof #$i of $TOTAL_PROOFS at $(date)"

    # Generate proof files. This part will be loud and may ask for entropy.
    (cd real-proof && \
     cat > input.json <<EOM
{ "a": "$((RANDOM % 10000))", "b": "$((RANDOM % 10000))" }
EOM
     snarkjs wtns calculate sum.wasm input.json witness.wtns && \
     snarkjs groth16 prove sum.zkey witness.wtns proof.json public.json && \
     mv proof.json public.json ../data/)

    # Build the JSON payload using jq
    JSON_PAYLOAD=$(jq -n \
      --argjson proof "$(cat data/proof.json)" \
      --argjson public "$(cat data/public.json)" \
      --argjson vk "$(cat data/verification_key.json)" \
      '{ proofType: "groth16", vkRegistered: false, proofOptions: { library: "snarkjs", curve: "bn128" }, proofData: { proof: $proof, publicSignals: $public, vk: $vk } }')

    # Submit proof with curl
    echo "Submitting proof with curl..."
    SUBMIT_RESPONSE=$(curl -s -X POST \
      -H "Content-Type: application/json" \
      -d "$JSON_PAYLOAD" \
      "${API_URL}/submit-proof/${API_KEY}")

    echo "$SUBMIT_RESPONSE" >> $LOG_FILE
    JOB_ID=$(echo "$SUBMIT_RESPONSE" | jq -r '.jobId')

    if [ -z "$JOB_ID" ] || [ "$JOB_ID" == "null" ]; then
        echo "❌ Failed to submit proof. See automation.log for details."
        JOB_ID="Submission Failed"
        TX_HASH="N/A"
    else
        echo "Proof submitted successfully. JobID: $JOB_ID"
        # Poll for status with curl
        while true; do
            STATUS_RESPONSE=$(curl -s "${API_URL}/job-status/${API_KEY}/${JOB_ID}")
            STATUS=$(echo "$STATUS_RESPONSE" | jq -r '.status')
            echo "   Polling... Current status: $STATUS"
            if [ "$STATUS" == "Finalized" ] || [ "$STATUS" == "Failed" ]; then
                echo "$STATUS_RESPONSE" >> $LOG_FILE
                TX_HASH=$(echo "$STATUS_RESPONSE" | jq -r '.txHash // "Failed"')
                break
            fi
            sleep 5
        done
    fi

    echo "✅ Result: JobID = $JOB_ID, TxHash = $TX_HASH"
    echo "$(date),${i},${JOB_ID},${TX_HASH}" >> $CSV_FILE
    echo "Waiting for 30 seconds..."
    sleep 30
done

echo "✅ Automation Complete."
EOF
```

Make the script executable:

```bash
chmod +x automate.sh
```

## 💻 Running the Automator
You are now ready to start submitting proofs automatically.

To Start the Script
It's highly recommended to run the script inside a screen session, which allows it to continue running even if you disconnect from your server.

Start a new screen session:

```bash
screen -S zkverify
```

Run the automation script:

```bash
./automate.sh
```

Detach from the session: To let it run in the background, press Ctrl+A, then press the d key. Note that you may need to re-attach to enter the random text for entropy if prompted.

To Check Your Results
Re-attach to the screen to see the live output at any time:

```bash
screen -r zkverify
```

View your collected JobIDs and TxHashes by reading the CSV file:

```bash
cat submissions.csv
```

## 🏆 Claim Your Points
Use the jobId or txHash from your submissions.csv file to fill out the official zkVerify submission form to claim your points for the testnet.

➡️ Submission Form: https://forms.gle/PVjhLkDt2TbgmspGA

Good luck!
