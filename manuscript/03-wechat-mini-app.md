# Building a WeChat Mini App with Tencent Cloud

This chapter provides a concise, end-to-end recipe for launching a production-ready WeChat Mini Program using Tencent Cloud's Serverless CloudBase stack. It follows the official "WeChat Mini Program + Serverless" reference architecture described in Tencent Cloud documentation and adapts it for an AI-first platform mindset.

## 1. Prerequisites
- **Accounts**: WeChat developer account, Tencent Cloud account with verified identity, and an initialized CloudBase environment.
- **Tooling**: Install the latest WeChat Developer Tools, enable the **Mini Program** mode, and sign in with your WeChat developer account.
- **Project scaffolding**: Either create a new Mini Program in the Developer Tools or clone an existing template. When prompted for **Cloud Development**, enable **Serverless CloudBase** so a default environment is created.

## 2. Recommended Architecture
- **Front-end**: Native Mini Program pages (WXML/WXSS/JS) that call cloud functions through the `wx.cloud` SDK.
- **Backend**: CloudBase cloud functions for business logic and scheduled jobs, with CloudBase Database for storage and CloudBase Storage for media.
- **Security**: Rely on WeChat's built-in authentication (`wx.cloud.callFunction({ name: 'login' })`) and apply per-collection rules in CloudBase Database to scope data to the authenticated `openid`.

## 3. Project Setup Steps
1. **Create the Mini Program project** in WeChat Developer Tools and bind it to your Tencent Cloud account. Confirm the auto-provisioned CloudBase environment (e.g., `env-xxxx`).
2. **Initialize cloud capabilities** in `app.js`:
   ```javascript
   App({
     onLaunch() {
       wx.cloud.init({
         env: 'env-xxxx', // replace with your environment ID
         traceUser: true,
       });
     },
   });
   ```
3. **Generate a login function** (`cloudfunctions/login/index.js`) using the built-in template:
   ```javascript
   const cloud = require('wx-server-sdk');
   cloud.init({ env: cloud.DYNAMIC_CURRENT_ENV });

   exports.main = async () => {
     const { OPENID, APPID } = cloud.getWXContext();
     return { openid: OPENID, appid: APPID };
   };
   ```
4. **Create a simple page** (`miniprogram/pages/index/index.js`) that calls the function and renders data:
   ```javascript
   Page({
     data: { user: null, todos: [] },
     async onLoad() {
       const res = await wx.cloud.callFunction({ name: 'login' });
       this.setData({ user: res.result });

       const { data } = await wx.cloud.database().collection('todos').get();
       this.setData({ todos: data });
     },
   });
   ```
5. **Define a collection rule** in CloudBase Database (Console → Database → Rules):
   ```
   // Only allow authenticated users to read/write their own documents
   { "read": "doc._openid == auth.openid", "write": "doc._openid == auth.openid" }
   ```
6. **Deploy cloud functions** from the Developer Tools (Cloud → Functions → Deploy) and **upload the Mini Program** for preview or production.

## 4. Example Data Model
- `todos` collection: `{ title: string, done: boolean, _openid: string, createdAt: timestamp }`
- `sessions` collection (optional): Use for feature flags or experimentation; scope reads by `auth.openid`.

## 5. Observability and Ops
- Enable **CloudBase Monitoring** for function duration, cold starts, and errors.
- Configure **CI/CD** by binding the project to a Git repository and enabling auto-deploy in the WeChat Developer Tools.
- Set up **alerts** in Tencent Cloud for function error rate and database QPS to catch regressions early.

## 6. AI-First Enhancements (Optional)
- Add a **content moderation** check by calling the `security.msgSecCheck` cloud API before saving user-generated text.
- Integrate a **vector search service** (e.g., via CloudBase functions calling external embeddings APIs) to personalize recommendations in the Mini Program.
- Use **scheduled cloud functions** for daily retraining jobs or to refresh recommendations.

## 7. Checklist for Release
- [ ] Replace `env-xxxx` with the actual CloudBase environment ID.
- [ ] Verify database rules with a test user and ensure no cross-tenant data leaks.
- [ ] Run the **Preview** build in WeChat Developer Tools on both iOS and Android devices.
- [ ] Submit for **WeChat audit**, including correct privacy disclosures and icon assets.
- [ ] Tag the release in Git and document deployment steps for your team.

With these steps, you can create, secure, and ship a functional WeChat Mini Program that leverages Tencent Cloud's serverless backend while keeping the door open for AI-driven features.
