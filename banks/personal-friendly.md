# 个人开发者友好的银行 API

以下列表分两类：
- **可直接用个人账户接入**：bunq、Monzo、Starling
- **个人可研究但标准个人账户不能直接拿 API key**：Revolut（需 Business 或聚合器）

---

## bunq

- **国家**: 🇳🇱 荷兰
- **文档**: https://doc.bunq.com/
- **发现索引**: https://doc.bunq.com/llms.txt
- **认证**: API Key + RSA 密钥对
- **个人可用**: ✅ 完整可用
- **特点**: 完全访问账户，无只读模式；标准 payment 可免 app 确认执行；提供 sandbox API 与 sandbox user/key 流程
- **实践状态**: 已验证生产集成，详见本仓库 README.md

---

## Monzo

- **国家**: 🇬🇧 英国
- **文档**: https://docs.monzo.com/
- **开发者门户**: https://developers.monzo.com/
- **认证**: OAuth 2.0
- **个人可用**: ✅ 只能连自己的账户或小范围授权用户
- **特点**:
  - 不适合构建公开应用（官方明确说明）
  - 需要 Monzo 个人账户才能申请 OAuth client
  - 支持读取余额、交易、创建 pot、发起付款
  - Webhook 支持实时通知
- **限制**: 仅限个人和小范围用户，不能对外发布产品

---

## Starling

- **国家**: 🇬🇧 英国
- **文档**: https://developer.starlingbank.com/docs
- **开发者门户**: https://developer.starlingbank.com/
- **GitHub 资源**: https://github.com/starlingbank/developer-resources
- **认证**: OAuth 2.0
- **个人可用**: ✅ 个人账户可直接注册开发者账号
- **特点**:
  - 提供 Sandbox 环境
  - 支持个人账户、联名账户、商业账户
  - API 覆盖余额、交易、付款、储蓄目标、卡片管理
  - 有活跃的开发者社区和第三方集成示例
- **限制**: 英国境内银行，需要英国地址开户

---

## Revolut

- **国家**: 🇬🇧 英国
- **文档**: https://developer.revolut.com/
- **Open Banking**: https://developer.revolut.com/docs/open-banking/open-banking-api
- **认证**: Business API / Merchant API 用密钥或证书；Open Banking 用 OAuth + eIDAS
- **个人可用**: ⚠️ 标准个人账户无直接 API key；Business 账户可直接接入
- **特点**:
  - **Open Banking API**: 面向 TPP，PSD2 合规
  - **Business API**: 企业账户可用，支持多币种、批量付款、卡管理
  - **Merchant API**: 收款和支付处理
  - 沙盒环境完整
  - 支持 30+ 货币
- **个人可用路径**:
  - 个人标准账户：主要走 Open Banking（需第三方聚合器如 TrueLayer/Plaid）
  - Revolut Business：可直接申请 Business API key
- **限制**: 个人标准账户没有直接 API key，需通过 Open Banking 聚合器或升级到 Business
