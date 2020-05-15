# HBTC Proof of Reserve

HBTC 资产存在证明

## 原理

对于一个接收用户充值的数字货币交易所来说，经常受到的一种质疑是：用户充值的币，是否还完好无损的保存在交易所的钱包里，有没有被盗，有没有被挪用，当用户提币的时候，能不能顺利的提走属于用户自己的币。

证明用户充值的币还完好无损的存在交易所钱包里的方案一般叫做"准备金证明"，步骤如下：

- 公布交易所的冷热钱包地址及余额
- 快照用户资产余额
- 利用用户资产余额快照，构造一棵 Merkle Tree，叶子为用户资产余额，根为用户资产总余额
- 公开 Merkle Tree 供用户验证

用户需要自主验证：

- 检查自己的余额是否存在于 Merkle Tree 叶子节点中
- 验证 Merkle Tree 从自己余额的叶子节点到根节点的计算，确认 hash 正确
- 对比根节点的资产总余额与平台公布的冷钱包钱包地址余额，确认交易所资产足够

## 局限性

- 即使证明了用户的币在交易所的钱包里，并不代表提币一定能提走。
- 需要有足够多的用户都来进行实际的验证操作，降低平台在 Merkle Tree 中伪造数据且不被发现的概率。理想情况下，所有持仓用户都执行过自助验证，且都验证正确，则证明平台生成的 Merkle Tree 是完全正确的。

## HBTC 的具体做法

### 原始数据选取

- 币种：HBTC 资产证明目前提供 BTC，ETH 和 USDT 三种资产证明。

- 余额快照：HBTC 选取2020年4月13号 UTC 时间 0 点（北京时间早上8点）的所有用户余额（汇总钱包账户、期权账户、合约账户及用户自定义子账户的余额），每个用户每个币种一条记录，包括 userId, nonce 随机码, amount 余额。本次 nonce 选择使用用户的钱包账户的 account id

- 冷热钱包地址:
  - BTC、USDT-OMNI:
    - 3Bm99oWTc7Eebrv4TEUBTY86t4AFXSdPRK
    - 378E5HP7SGcW6XvriEirgvNUF54G79mftB
    - 3CTgNAMCY6ZoMFYPNFo8xBLeVv4stcXBhx
  - ETH、USDT-ERC20:
    - 0x3c047B9Dfa4fd8F812f264f1611b959a4DD980f8
    - 0xA31974408AAD18C1e5C4648d435795Fa757EA47f
    - 0x1Ee7C37EFF6a37d38d805A0ceDc134147a9DD3Cf
    - 0xa4938E606eDfe350D08813a0785Ed85D4640d365
    - 0xCe76a14dABe64acb8b044c489374C8ca1f456837
  - USDT-TRC20:
    - TGR3kYhfAsxFGkAyWLRegaVSTXQ3Y36tfM

备注：

- ETH 和 USDT 因钱包归集策略原因，有部分资金未归集到热钱包地址
- 因币核云的托管清算平台特性，和流动性共享功能，HBTC 交易所的余额变动较快，部分余额变动未反应在链上热钱包余额上，而是由托管清算平台内部完成清算

### 叶子节点生成

- 将用户余额 amount * 10^8 并取整
- 计算 hash = sha256(userId + "|" + nonce + "|" + amount)
- 将 hash 转变为 16 进制字符串，并取前 16个字符，作为 hexHash，存储到节点上

### 树节点生成

- 因需要每 2 个子节点汇聚一个父节点，所以如果子节点数量为奇数时，将最后一个节点拷贝一份，amount 清零，作为 padding 节点使用
- 父节点 amount = 子节点 amount 之和
- 父节点 hash = sha256(父节点amount + "|" + leftNode.hexHash + "|" + rightNode.hexHash)
- 父节点 hexHash 为父节点 hash 转 16 进制，并取前 16个字符

### 用户验证子树生成

- 从根节点开始到用户所在叶子节点的唯一路径经过的所有节点，以及计算涉及到的其它节点的集合，称为用户验证子树。提供给用户用来做验证操作
- 用户自己的叶子节点称为 自己节点 self。旁边的另一个用户节点称为 用户节点 user
- 一级节点称为根节点 root
- 从叶子节点往上直到根节点，经过的节点称为相关节点 relevant
- 相关节点的计算涉及到的非相关节点，称为 普通节点 node

### 产品化

HBTC 在 PC 端网站上提供了供用户自助验证的页面 https://www.hbtc.com/finance/trace_back/BTC

## 数据示例

- BTC 示例

```json
```

- ETH 示例

```json
```

- USDT 示例

```json
```

## 验证办法

- 访问个人中心获取自己的 userId
- 访问资产页面获取自己的对应币种资产余额，将余额乘以 10^8 然后取整，作为 amount
- 访问资产证明页面，获取自己的 nonce
- 计算 sha256(userId + "|" + nonce + "|" + amount) 左边 16 bytes，与资产证明页面的 self 节点的 hash 比对
- 从 self 节点一直向上，计算验证每个节点的 hash : 首先将两个子节点的 amount 相加，作为父节点的 amount，然后计算 sha256(amount + "|" + leftNode.hexHash + "|" + rightNode.hexHash) 的左边 16 byte，与父节点的 hash 比对
- 一直计算到根节点，确认中间每一级的 amount 和 hash 都正确匹配

## 参考

- [HBTC Proof 生成和验证代码示例](https://github.com/bhexofficial/asset-proof)
- [proving-bitcoin-reserves](https://iwilcox.me.uk/2014/proving-bitcoin-reserves)
- [人人比特 100%准备金证明](https://github.com/RenrenBit/ProofOfReserves)
- [币付宝 100%准备金证明](https://www.bifubao.com/2014/03/16/proof-of-reserves/)
- [Proof of Liabilities (PoL)](http://syskall.com/proof-of-liabilities/) [Github Code](https://github.com/olalonde/proof-of-liabilities)
