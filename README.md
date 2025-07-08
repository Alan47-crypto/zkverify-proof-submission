# 🔐 zkVerify Automated Proof Submission Guide (Groth16)

A complete, working guide to automatically generate and submit thousands of unique ZK proofs to the [zkVerify Incentivized Testnet](https://points.zkverify.io/loyalty%3Freferral_code%3DKUO7D211) using **Circom**, **SnarkJS**, and a powerful automation script.

This guide provides a full end-to-end workflow, from installing the necessary tools to running a background script that handles everything for you.

---

## 📋 Prerequisites

Before you begin, make sure you have the following installed on your server (e.g., a Linux VPS):

- **Node.js**: Version 18+ is required (for `snarkjs`)
- **Git**, **curl**, and **jq**: Standard command-line tools
- **API Key**: You’ll need an API key from zkVerify (see Step 5)

---

## 🚀 End-to-End Setup and Automation

Follow these steps to set up the project and start the automation script.

---

### 🔧 Step 1: Install Global Tools

```bash
npm install -g circom
npm install -g snarkjs
```

---

### 📁 Step 2: Set Up Project Folders

```bash
# Create the main project folder and enter it
mkdir zkverify-automator && cd zkverify-automator

# Create sub-folders for proofs and data
mkdir real-proof data
```

---

### 🧮 Step 3: Create the ZK Circuit

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

### 🔐 Step 4: One-Time Key Generation Ceremony

```bash
cd real-proof

# 1. Compile the circuit
circom sum.circom --r1cs --wasm --sym -o .

# 2. Start the Powers of Tau ceremony
snarkjs powersoftau new bn128 12 pot12_0000.ptau -v

# 3. Contribute randomness (entropy)
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="My Contribution" -v

# 4. Prepare for the final phase
snarkjs powersoftau prepare phase2 pot12_0001.ptau pot12_final.ptau -v

# 5. Generate the final proving key (.zkey)
snarkjs groth16 setup sum.r1cs pot12_final.ptau sum.zkey

# 6. Export the verification key to JSON
snarkjs zkey export verificationkey sum.zkey verification_key.json
```

---

### 🔑 Step 5: Set Your API Key

```bash
# Navigate back to project root
cd ..

# Create .env file with placeholder
echo "API_KEY=YOUR_API_KEY_HERE" > .env

# Edit it to paste your actual API key
nano .env
```

➡️ **Get your key from the [zkVerify Discord](https://discord.gg/zkverify)**

---

### 🤖 Step 6: Create the Final Automation Script

```bash
cat << 'EOF' > automate.sh
#!/bin/bash
# FINAL, WORKING AUTOMATION SCRIPT

# --- CONFIGURATION ---
TOTAL_PROOFS=1000
LOG_FILE="automation.log"
CSV_FILE="submissions.csv"
SLEEP_INTERVAL=30

# --- SCRIPT START ---

echo "✅ Starting zkVerify Fully Automated Script..."
if [ ! -f "$CSV_FILE" ]; then
    echo "Timestamp,Proof_Number,JobID,TxHash,Status" > "$CSV_FILE"
fi
API_KEY=$(grep API_KEY .env | cut -d '=' -f2)
API_URL="https://relayer-api.horizenlabs.io/api/v1"

if ! command -v jq &> /dev/null; then
    echo "jq could not be found. Please install it with 'sudo apt-get install jq'"
    exit 1
fi

if [ -f "real-proof/verification_key.json" ]; then
    mv real-proof/verification_key.json data/
fi

for i in $(seq 1 $TOTAL_PROOFS)
do
    echo "--------------------------------------------------"
    echo "➡️  Starting Proof #$i of $TOTAL_PROOFS at $(date)"
    
    (cd real-proof && \
     cat > input.json <<EOM
{ "a": "$((RANDOM % 10000))", "b": "$((RANDOM % 10000))" }
EOM
     snarkjs wtns calculate sum.wasm input.json witness.wtns && \
     echo "$(head -c 20 /dev/urandom | tr -dc 'a-zA-Z0-9')" | snarkjs groth16 prove sum.zkey witness.wtns proof.json public.json && \
     mv proof.json public.json ../data/) &> /dev/null

    JSON_PAYLOAD=$(jq -n \
      --argjson proof "$(cat data/proof.json)" \
      --argjson public "$(cat data/public.json)" \
      --argjson vk "$(cat data/verification_key.json)" \
      '{ proofType: "groth16", vkRegistered: false, proofOptions: { library: "snarkjs", curve: "bn128" }, proofData: { proof: $proof, publicSignals: $public, vk: $vk } }')

    SUBMIT_RESPONSE=$(curl -s -X POST \
      -H "Content-Type: application/json" \
      -d "$JSON_PAYLOAD" \
      "${API_URL}/submit-proof/${API_KEY}")
    
    echo "$SUBMIT_RESPONSE" >> $LOG_FILE
    JOB_ID=$(echo "$SUBMIT_RESPONSE" | jq -r '.jobId')

    if [ -z "$JOB_ID" ] || [ "$JOB_ID" == "null" ]; then
        JOB_ID="Submission Failed"
        TX_HASH="N/A"
        FINAL_STATUS="Failed"
    else
        echo "   Proof submitted. JobID: $JOB_ID"
        while true; do
            STATUS_RESPONSE=$(curl -s "${API_URL}/job-status/${API_KEY}/${JOB_ID}")
            STATUS=$(echo "$STATUS_RESPONSE" | jq -r '.status // empty')

            if [ -z "$STATUS" ]; then
                echo "   ⚠️ Warning: Empty status response, retrying..."
                sleep 5
                continue
            fi
            
            echo "   ⏳ Status: $STATUS"
            
            if [[ "$STATUS" == "Finalized" || "$STATUS" == "Failed" || "$STATUS" == "IncludedInBlock" ]]; then
                TX_HASH=$(echo "$STATUS_RESPONSE" | jq -r '.txHash // "No TxHash Yet"')
                FINAL_STATUS=$STATUS
                break
            fi
            sleep 5
        done
    fi
    
    echo "   ✅ Result: JobID = $JOB_ID, TxHash = $TX_HASH, Status = $FINAL_STATUS"
    echo "$(date),${i},${JOB_ID},${TX_HASH},${FINAL_STATUS}" >> $CSV_FILE
    echo "   Waiting ${SLEEP_INTERVAL} seconds..."
    sleep ${SLEEP_INTERVAL}
done

echo "✅ Automation Complete."
EOF
```

---

### 🔓 Step 7: Make the Script Executable and Run It

```bash
chmod +x automate.sh
./automate.sh
```

💡 **Tip:** Use `screen` or `tmux` to run the script in the background for long sessions.

---

## 🏆 Claim Your Points

Once the script runs, your proof results will be stored in `submissions.csv`.

Use the JobID or TxHash from the file to fill out the zkVerify form and claim your testnet points.

➡️ [Submit here](https://forms.gle/PVjhLkDt2TbgmspGA)
