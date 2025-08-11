## Poseidon2 哈希电路实现方案（Circom + Groth16）

### 1. 算法与参数

* **算法类型**：Poseidon2（基于海绵结构）
* **状态宽度**：`t = 3`（2 个输入 + 1 容量位）
* **位宽**：`n = 256`
* **S-box 指数**：`d = 5`
* **轮数参数**：

  * **完整轮数（RF\_FULL）**：8（首尾各 4 轮）
  * **部分轮数（RF\_PARTIAL）**：56（中间部分）
  * **总轮数**：64（8 + 56）

---

### 2. 电路输入/输出设计

* **隐私输入（private）**
  `in_private[2]`：两个 256 位字段元素（原像）
* **公开输入（public）**
  `hash`：256 位 Poseidon2 哈希值

---

### 3. Poseidon2 置换结构

1. **初始化状态**

   $$
   state = [in\_private[0],\ in\_private[1],\ 0]
   $$
2. **每轮处理**：

   * **ARK（Add Round Key）**：加轮常数
   * **S-box**：非线性变换 $x^5$
   * **MDS（Maximum Distance Separable）**：线性扩散层
3. **输出**

   * 哈希值为 `state[0]`

---

### 4. 电路模块划分

#### 4.1 SBox.circom

* 功能：计算 $x^5$
* 用于完整轮和部分轮（部分轮只对一个元素做 S-box）

#### 4.2 Poseidon2Round.circom

* 功能：实现单轮的 ARK → S-box → MDS
* 轮类型：

  * 完整轮：所有元素做 S-box
  * 部分轮：仅第一个元素做 S-box

#### 4.3 Poseidon2Hash.circom

* 功能：Poseidon2 哈希主电路
* 步骤：

  1. 初始化状态
  2. 循环执行 64 轮置换
  3. 输出 `state[0]`
  4. 添加 `===` 约束保证输出与公开输入一致

---

### 5. Groth16 集成流程

#### 步骤：

1. 编写 Circom 电路（Poseidon2Hash）
2. 编译生成 R1CS 和 WASM
3. **可信设置**（Trusted Setup）
4. 生成证明密钥（`pk`）和验证密钥（`vk`）
5. 计算 Witness
6. 使用 `groth16.prove` 生成证明
7. 使用 `groth16.verify` 验证证明

#### 流程图：

```
graph TD
    A[编写 Circom 电路] --> B[编译电路]
    B --> C[生成 R1CS 和 WASM]
    C --> D[可信设置]
    D --> E[生成证明密钥 pk 和验证密钥 vk]
    E --> F[计算 Witness]
    F --> G[生成证明 Proof]
    G --> H[验证证明]
```

---

### 6. 测试脚本示例（Node.js）

```javascript
const { wasm: wasmTester } = require("circom_tester");
const { groth16 } = require("snarkjs");
const { poseidon } = require("circomlibjs");

async function main() {
    // 1. 编译电路
    const circuit = await wasmTester("poseidon2.circom");

    // 2. 准备输入
    const inputs = {
        in_private: [
            "12345678901234567890123456789012",
            "98765432109876543210987654321098"
        ]
    };

    // 3. 计算参考哈希
    const hash = await poseidon(inputs.in_private);
    inputs.hash = hash.toString();

    // 4. 生成见证
    const witness = await circuit.calculateWitness(inputs);

    // 5. Groth16 设置
    const zkey = await groth16.setup(circuit);

    // 6. 生成证明
    const { proof, publicSignals } = await groth16.prove(zkey, witness);

    console.log("Proof:", proof);
    console.log("Public Signals:", publicSignals);

    // 7. 验证证明
    const verificationKey = await groth16.exportVerificationKey(zkey);
    const isValid = await groth16.verify(verificationKey, publicSignals, proof);
    console.log("Verification:", isValid ? "SUCCESS" : "FAILED");
}

main().catch(console.error);
```

---

### 总结

该方案使用 Circom 实现 Poseidon2 哈希电路，采用 (n,t,d) = (256,3,5) 参数，并将哈希值作为公开输入，原像作为隐私输入，通过 Groth16 生成可验证的零知识证明，流程包括电路编写、编译、可信设置、见证计算、证明生成与验证。

---
