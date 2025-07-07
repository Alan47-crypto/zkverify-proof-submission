# 🔐 zkVerify Automated Proof Submission Guide (Groth16)

✅ A complete, working guide to automatically generate and submit thousands of unique ZK proofs to the zkVerify Incentivized Testnet using Circom, SnarkJS, and a powerful automation script.

This guide provides a full end-to-end workflow, from installing the necessary tools to running a background script that handles everything for you.

---

## 📋 Prerequisites

Before you begin, make sure you have the following installed on your server (e.g., a Linux VPS):

- **Node.js:** Version 18+ is required.
- **API Key:** You'll need an API key from zkVerify. See Step 5 for instructions.

---

## 🚀 End-to-End Setup and Automation

Follow these steps to set up the project and start the automation script.

### Step 1: Install Global Tools

First, install circom and snarkjs globally using npm.

```bash
npm install -g circom
npm install -g snarkjs
```

---

### Step 2: Set Up Project Folders

Create the directory structure for the project.

```bash
# Create the main project folder and enter it
mkdir zkverify-automator && cd zkverify-automator

# Create sub-folders for proofs and data
mkdir real-proof data
```

---

### Step 3: Create the ZK Circuit

Next, create the simple `sum.circom` circuit file. This circuit will add two numbers, `a` and `b`.

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

---

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

---

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

> **Important:** To have your submissions tracked for the incentivized testnet and earn points, you must use your own unique API key.  
> ➡️ Request your key here: [zkVerify Discord](https://discord.gg/zkverify)

Once you have your key, create the environment file by running the two commands below.

```bash
echo "API_KEY=YOUR_API_KEY_HERE" > .env
```

Edit the file and paste your key:

```bash
nano .env
```

Replace `YOUR_API_KEY_HERE` with the actual key you received, then save and exit (`Ctrl+X`, then `Y`, then `Enter`).

#### 🧪 For Quick Testing (Optional)

If you just want to run a quick test without your submissions being tracked for points, you can use the default public API key:

```bash
echo "API_KEY=598f259f5f5d7476622ae52677395932fa98901f" > .env
```

---

#### Create the Node.js Submission Script (`index.js`)

This command creates the script file that sends your proof to the relayer.

```bash
cat << 'EOF' > index.js
import axios from 'axios';
import fs from 'fs';
import dotenv from 'dotenv';
dotenv.config();

const API_URL = 'https://relayer-api.horizenlabs.io/api/v1';

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

---

### Step 6: Create the Final Automation Script

This script ties everything together, running the process in a loop to submit many unique proofs.

#### Check for jq

This tool can help parse results, but the final script uses a more basic method. It's still good to have.

```bash
# Check version
jq --version

# Install on Debian/Ubuntu if needed
sudo apt-get update && sudo apt-get install -y jq
```

#### Create `automate.sh`:

```bash
cat << 'EOF' > automate.sh
#!/bin/bash

# --- CONFIGURATION ---
TOTAL_PROOFS=1000  # Set the total number of proofs to submit
LOG_FILE="automation.log"
CSV_FILE="submissions.csv"

# --- SCRIPT START ---

echo "Starting zkVerify Automation Script..."
if [ ! -f "$CSV_FILE" ]; then
    echo "Timestamp,Proof_Number,JobID,TxHash" > $CSV_FILE
fi
echo "Results will be saved to $CSV_FILE"

# Move the one-time verification key
if [ -f "real-proof/verification_key.json" ]; then
    mv real-proof/verification_key.json data/
fi

for i in $(seq 1 $TOTAL_PROOFS)
do
    echo "--------------------------------------------------"
    echo "➡️ Starting Proof #$i of $TOTAL_PROOFS at $(date)"

    # Generate and move proof files silently
    (cd real-proof && \
     cat > input.json <<EOM
{
  "a": "$((RANDOM % 10000))",
  "b": "$((RANDOM % 10000))"
}
EOM
     snarkjs wtns calculate sum.wasm input.json witness.wtns && \
     snarkjs groth16 prove sum.zkey witness.wtns proof.json public.json && \
     mv proof.json public.json ../data/) &> /dev/null

    # Run submission script, capturing both stdout and stderr
    OUTPUT=$(node index.js 2>&1)

    # Log the raw output
    echo "--- Run #$i Output ---" >> $LOG_FILE
    echo "$OUTPUT" >> $LOG_FILE

    # Use robust grep and sed for parsing
    JOB_ID=$(echo "$OUTPUT" | grep -o '"jobId":"[^"]*' | sed 's/"jobId":"//' | head -n 1)
    TX_HASH=$(echo "$OUTPUT" | grep -o '"txHash":"[^"]*' | sed 's/"txHash":"//')

    if [ -z "$JOB_ID" ]; then JOB_ID="Not Found"; fi
    if [ -z "$TX_HASH" ]; then TX_HASH="Pending or Failed"; fi

    echo "✅ Result: JobID = $JOB_ID, TxHash = $TX_HASH"

    # Save results to CSV
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

---

## 💻 Running the Automator

You are now ready to start submitting proofs automatically.

### To Start the Script

It's highly recommended to run the script inside a `screen` session, which allows it to continue running even if you disconnect from your server.

Start a new screen session:

```bash
screen -S zkverify
```

Run the automation script:

```bash
./automate.sh
```

Detach from the session: To let it run in the background, press `Ctrl+A`, then press the `d` key.

---

### To Check Your Results

Re-attach to the screen to see the live output at any time:

```bash
screen -r zkverify
```

View your collected JobIDs and TxHashes by reading the CSV file:

```bash
cat submissions.csv
```

---

## 🏆 Claim Your Points

Use the jobId or txHash from your `submissions.csv` file to fill out the official zkVerify submission form to claim your points for the testnet.

➡️ [Submission Form](https://forms.gle/PVjhLkDt2TbgmspGA)

---

Good luck!
