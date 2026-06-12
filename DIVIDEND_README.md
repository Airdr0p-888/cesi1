# FairMint Token 分红合约使用说明

## 概述

从本版本开始，持币分红功能由独立的 `DividendDistributor` 合约处理，而非主合约内置逻辑。这样设计的好处：

1. **资金安全**：分红 BNB 留在独立合约中，即使主合约丢弃权限，项目方仍可通过分红合约管理
2. **职责分离**：主合约负责税收分配，分红合约专责分红分配
3. **可升级性**：分红合约可独立升级或迁移

## 部署

分红合约在主线合约部署时**自动部署**，无需单独操作。

构造函数中：
```solidity
// 部署独立分红合约
// admin 设置为营销钱包，即使主合约丢弃权限，分红合约仍可由项目方管理
dividendDistributor = new DividendDistributor(marketingWallet);
```

## 查询分红合约地址

调用主合约的 `dividendDistributor()` 函数：
```javascript
const dividendDistributorAddress = await fairMintToken.dividendDistributor();
console.log("分红合约地址:", dividendDistributorAddress);
```

## 持币人领取分红

### 方式一：直接调用分红合约（推荐）
```javascript
// 直接调用分红合约的 claim() 函数
await dividendDistributor.claim();
```

### 方式二：通过主合约代理
```javascript
// 调用主合约的 claimDividend() 函数（内部调用分红合约）
await fairMintToken.claimDividend();
```

## 查询可领取分红

调用主合约的 `dividendOf(address)` 函数（内部查询分红合约）：
```javascript
const claimable = await fairMintToken.dividendOf(userAddress);
console.log("可领取分红:", ethers.formatEther(claimable), "BNB");
```

或直接查询分红合约：
```javascript
const claimable = await dividendDistributor.getClaimableDividend(userAddress);
```

## 税收分配流程

1. 用户交易时，主合约扣除税费（买入税/卖出税）
2. 税费按比例分配：
   - **营销部分**：直接兑换 BNB 发送给营销钱包
   - **销毁部分**：直接销毁代币
   - **分红部分**：兑换 BNB 后发送给分红合约（`dividendDistributor.deposit{value: bnbReceived}()`）
   - **回流 LP 部分**：一半兑换 BNB，添加流动性
3. 分红合约收到 BNB 后，更新 `dividendsPerShare`（每份额分红）
4. 持币人主动 `claim` 领取分红

## 管理员功能

分红合约的 `admin` 默认为营销钱包地址，拥有以下权限：

### 1. 提取滞留 BNB
长时间无人领取的分红，管理员可提取用于其他用途（需社区共识）：
```javascript
await dividendDistributor.withdrawStuckBNB(toAddress, amount);
```

### 2. 更新管理员地址
```javascript
await dividendDistributor.setAdmin(newAdminAddress);
```

### 3. 紧急提取所有 BNB
用于迁移或升级分红合约：
```javascript
await dividendDistributor.emergencyWithdraw();
```

### 4. 批量代领（可选）
为多个持币人批量领取分红（消耗大量 gas，需谨慎）：
```javascript
await dividendDistributor.claimFor([address1, address2, ...]);
```

## 主合约集成细节

### 转账时更新份额
主合约的 `_update` 函数中，每次转账后会调用分红合约更新持币人份额：
```solidity
// 更新分红合约中的持币人份额（使用转账后的最新余额）
if (from != address(0) && from != address(this)) {
    dividendDistributor.setShare(from, balanceOf(from));
}
if (to != address(0) && to != address(this)) {
    dividendDistributor.setShare(to, balanceOf(to));
}
```

### `setShare` 函数逻辑
```solidity
function setShare(address shareholder, uint256 balance) external onlyToken {
    uint256 oldBalance = shares[shareholder];
    
    if (oldBalance > 0) {
        // 先计算应得分红
        uint256 claimable = getClaimableDividend(shareholder);
        if (claimable > 0) {
            claimedDividends[shareholder] += claimable;
            // 发送 BNB 给持币人
            (bool success, ) = payable(shareholder).call{value: claimable}("");
            if (!success) {
                claimedDividends[shareholder] -= claimable; // 回滚
            }
        }
    }
    
    // 更新份额
    shares[shareholder] = balance;
    // 重新计算 totalShares...
}
```

**注意**：`setShare` 会触发 BNB 转账，确保 `ReentrancyGuard` 防护。

## 安全考虑

1. **重入攻击**：分红合约的 `setShare` 和 `claim` 函数应添加 `nonReentrant` 修饰符（当前版本未添加，需在后续版本优化）
2. **管理员权限**：`admin` 可以提取滞留 BNB，需确保是可信地址（建议使用多签钱包）
3. **主合约权限**：`onlyToken` 修饰符确保只有主合约可以调用 `setShare` 和 `deposit`

## 测试建议

1. 部署后调用 `dividendDistributor()` 确认分红合约地址
2. 执行几笔交易，触发税费分配
3. 调用 `dividendOf(user)` 查看用户可领取分红
4. 调用 `claim()` 领取分红，确认 BNB 到账
5. 测试管理员功能（`withdrawStuckBNB`、`setAdmin` 等）

## 常见问题

### Q: 主合约丢弃权限后，分红合约还能正常工作吗？
**A**: 能！分红合约有独立的 `admin`，即使主合约 `renounceOwnership()`，分红合约仍可由 `admin` 管理。

### Q: 分红合约的 BNB 会不会丢失？
**A**: 不会。所有 BNB 都在分红合约中，持币人可随时 `claim`。即使无人领取，管理员也可提取（需社区共识）。

### Q: 持币人需要 gas 来领取分红吗？
**A**: 需要。持币人调用 `claim()` 需要消耗 gas。建议分红累计到一定金额后再领取。

### Q: 可以取消独立分红合约，改回主合约内置分红吗？
**A**: 可以，但需要修改主合约代码并重新部署。建议在测试网充分测试后再部署到主网。
