# FairMint Token - Mint 指南

## 重要提示 ⚠️

**用户必须通过调用 `mint()` 函数来 Mint 代币，不能直接向合约转 BNB！**

如果你直接向合约转 BNB（不调用函数），交易会失败并被拒绝。这是为了保护你的资金安全，防止在 Mint 满后误转 BNB。

## Mint 方式

### 方式 1：使用 Mint 页面（推荐）

1. 部署成功后，打开 `mint-ui.html` 页面
2. 输入合约地址（或直接在 URL 中添加参数：`mint-ui.html?contract=0x你的合约地址`）
3. 连接钱包
4. 点击 "Mint" 按钮，确认交易
5. 等待交易确认，代币会自动转入你的钱包

### 方式 2：通过 BSCScan 调用 `mint()` 函数

1. 打开 BSCScan（主网：https://bscscan.com，测试网：https://testnet.bscscan.com）
2. 搜索你的合约地址
3. 进入 "Contract" → "Write Contract"
4. 连接 Web3 钱包（MetaMask 或 Trust Wallet）
5. 找到 `mint()` 函数
6. 输入 Value（BNB 数量，如 0.01），点击 "Write"
7. 确认交易

### 方式 3：通过项目官网 Mint

如果你的项目有官网，通常会有 Mint 按钮，点击即可自动调用 `mint()` 函数。

## 常见问题

### Q: 为什么不能直接向合约转 BNB？

A: 为了防止用户误操作。如果 Mint 满了，用户直接转 BNB 会导致资金丢失（因为合约不会返回代币）。通过调用 `mint()` 函数，合约会检查 Mint 状态，如果已满会拒绝交易并退回 BNB。

### Q: 如果我不小心直接向合约转了 BNB 怎么办？

A: 交易的 `receive()` 函数会拒绝，你的 BNB 不会转入合约，交易会失败。你可以在钱包中看到失败的交易，但没有资金损失。

### Q: Mint 满后还能转 BNB 吗？

A: 不能。Mint 满后，`mint()` 函数会拒绝，直接转 BNB 也会被 `receive()` 函数拒绝。

### Q: 为什么合约还能接收 BNB？

A: 合约的 `receive()` 函数允许以下情况接收 BNB：
- PancakeSwap Router 在税收兑换时向合约发送 BNB
- 分红合约向合约转 BNB
- Owner（部署者）向合约转 BNB（用于测试或紧急操作）

这些情况都是安全的，不会导致用户资金损失。

## 技术细节

### 合约的 `receive()` 函数

```solidity
receive() external payable {
    // 允许的情况：
    // 1. Router 转 BNB（税收兑换）
    // 2. 分红合约转 BNB（用户提取分红）
    // 3. Owner 转 BNB（紧急操作）
    // 禁止：普通用户直接转 BNB
    if (msg.sender != address(uniswapV2Router) && 
        msg.sender != address(dividendDistributor) && 
        msg.sender != owner()) {
        revert("Direct BNB transfer not allowed, please call mint()");
    }
}
```

### 用户调用 `mint()` 函数

```solidity
function mint() external payable nonReentrant {
    require(mintActive, "Mint is not active");
    require(mintedSlots < totalMintSlots, "All slots minted");
    require(msg.value == mintPrice, "Incorrect BNB amount");
    require(!hasMinted[msg.sender], "Already minted");

    if (whitelistEnabled) {
        require(whitelist[msg.sender], "Not in whitelist");
    }

    hasMinted[msg.sender] = true;
    mintedSlots++;

    // 转移代币给用户
    _transfer(address(this), msg.sender, mintCount);

    // Mint 完成后自动添加流动性
    if (mintedSlots >= totalMintSlots) {
        _addInitialLiquidity();
    }
}
```

注意：用户调用 `mint()` 函数时，BNB 是作为 `msg.value` 传递的，这些 BNB 会留在合约中用于添加流动性。这不会触发 `receive()` 函数，因为 `msg.data` 不为空（有函数调用数据）。

## 安全建议

1. **始终使用 `mint()` 函数**：不要直接向合约转 BNB
2. **检查 Mint 状态**：在 Mint 前，先检查 Mint 是否已满
3. **使用官方前端**：如果有官网，优先使用官网的 Mint 功能
4. **小心钓鱼网站**：确保你访问的是正确的合约地址和官网

## 联系支持

如果你在 Mint 过程中遇到问题，请联系项目方或查看项目文档。
