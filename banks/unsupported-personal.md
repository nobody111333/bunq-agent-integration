# 不支持个人直接调用的银行 API

以下银行虽有 Developer Portal 或 Open Banking 接口，但**个人开发者无法直接申请 API key 调用自己的账户**。它们主要面向注册 TPP（Third Party Provider）或企业客户。

---

## 原因说明

PSD2 法规要求这些银行仅向持牌第三方服务商（AISP/PISP）开放账户数据接口。个人获取访问权限的门槛：

- 在欧盟/英国金融监管机构注册为 TPP
- 持有 eIDAS 合格证书（QWAC/QSealC）
- 通过柏林集团（Berlin Group）或 STET 等标准协议认证

这些流程成本高昂、周期数月，不适合个人 agent 集成场景。

---

## 银行列表

| 银行 | 国家 | Developer Portal | 个人直接可用 | 原因 |
|------|------|-----------------|-------------|------|
| **ING** | 🇳🇱 荷兰 | https://developer.ing.com/ | ❌ | 仅 PSD2，需 TPP 注册 |
| **Deutsche Bank** | 🇩🇪 德国 | https://developer.db.com/ | ❌ | 仅 PSD2，需 TPP 注册 |
| **BBVA** | 🇪🇸 西班牙 | https://www.bbvaapimarket.com/ | ❌ | 仅 PSD2，需 TPP 注册 |
| **Santander** | 🇬🇧 英国 | https://developer.santander.co.uk/ | ❌ | 仅 PSD2，需 TPP 注册 |
| **N26** | 🇩🇪 德国 | —（通过 support 页面） | ❌ | 仅 PSD2，无个人开发者门户 |
| **ABN AMRO** | 🇳🇱 荷兰 | https://developer.abnamro.com/ | ❌ | 仅 PSD2，需 TPP 注册 |
| **Nordea** | 🇫🇮 北欧 | https://www.nordea.com/en/our-services/nordea-api-market | ❌ | 仅 PSD2/企业 API，需注册 |
| **Danske Bank** | 🇩🇰 丹麦 | https://developers.danskebank.com/ | ❌ | 仅 PSD2/Premium API，需 TPP 或企业合同 |
| **La Banque Postale** | 🇫🇷 法国 | https://developer.labanquepostale.com/ | ❌ | 仅 PSD2，需 TPP 注册 |
| **Rabobank** | 🇳🇱 荷兰 | https://developer.rabobank.com/ | ❌ | 仅 PSD2，需 TPP 注册 |
| **BNP Paribas** | 🇫🇷 法国 | https://developers.cib.bnpparibas.com/ | ❌ | 面向企业和投行客户 |
| **Société Générale** | 🇫🇷 法国 | https://developer.societegenerale.fr/ | ❌ | 主要面向企业开发者 |

---

## 个人替代方案

若需访问这些银行的账户数据，个人用户可考虑：

1. **Open Banking 聚合器**（如 Plaid、Truelayer、Tink、GoCardless）
   - 它们已注册为 TPP，个人可通过其 SDK 间接访问
   - 通常有免费 tier，按请求计费
   - 缺点：多一层中间商，数据延迟

2. **银行官方 App 的导出功能**
   - 多数银行支持 CSV/PDF 导出
   - 可配合 agent 的文档解析工具使用
   - 非实时，但零门槛
