流程如下：

```
1. 用户登录 → 服务端返回 AccessToken(30min) + RefreshToken(7天)
2. 用户正常请求，携带 AccessToken → 网关验证通过 → 放行
3. 30分钟后 AccessToken 过期 → 请求返回 401
4. 前端检测到 401 → 自动调用刷新接口，携带 RefreshToken
5. 服务端验证 RefreshToken 有效 → 查询用户信息 → 生成新的 AccessToken(新30min) 返回
6. 前端用新 AccessToken 重试原请求 → 用户无感知
```

关键点：

- **RefreshToken 不在业务请求中传递**，只在刷新接口使用，减少泄露风险
- **RefreshToken 有效期长（7天）**，用户在这期间无需重新登录
- **RefreshToken 只包含用户ID**，不包含敏感信息，即使泄露影响更可控
- **AccessToken 短时效（30min）**，即使被截获，攻击者可用窗口短

文档中的对应代码：

```java
// 生成 AccessToken（含用户信息）
String accessToken = JwtUtils.generateToken(user.getId(), user.getUsername(), user.getRole());
// 生成 RefreshToken（仅含用户ID）
String refreshToken = JwtUtils.generateRefreshToken(user.getId());
```