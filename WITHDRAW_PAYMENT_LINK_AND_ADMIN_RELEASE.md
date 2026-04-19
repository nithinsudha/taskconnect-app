# Withdraw: Payment Link at Withdraw Time + Admin Approve → Release

This doc explains how **Razorpay Payment Links** are used at withdraw time, where things appear in Razorpay, how **razorpayPaymentId** is set, and how **admin approve → payment release** works.

---

## 1. Flow overview

1. **Helper** goes to Wallet → Withdraw → fills amount and bank details → taps **Submit Payout Request**.
2. **On that button tap**, the app:
   - Calls **Razorpay Payment Links API** (`POST /v1/payment_links`) and creates a link for the withdraw amount, with `reference_id` = payout `requestId`.
   - Saves the payout request in Firestore with:
     - `razorpayPaymentLinkId` (e.g. `plink_xxx`)
     - `paymentLinkUrl` (short URL from Razorpay)
     - Bank details, amount, `status: 'pending'`, etc.
   - Deducts helper’s coins and increases **pendingPayout**.
3. **Where the link shows in Razorpay:** The created payment link appears in the Razorpay Dashboard here:  
   **https://dashboard.razorpay.com/app/paymentlinks**  
   (Dashboard → **Payment Links**). You’ll see the link with amount, reference_id (= requestId), status (e.g. issued/paid/cancelled).
4. **When the payment link is paid** (if you use it to collect payment): Razorpay sends callback/webhook with **razorpay_payment_id**. Your backend can then set **razorpayPaymentId** on the payout request in Firestore.
5. **Payment release:** When **admin approves**, your backend calls **Razorpay Payouts API** to send money to the helper’s bank, then **sets** the payout request in Firestore: `status: 'processed'`, `razorpayPayoutId`, `processedAt`, and reduces the user’s **pendingPayout**. That is how **payment release** is recorded for this withdraw flow.

---

## 2. Where things are set and stored

| What | When / where set | Where stored |
|------|-------------------|--------------|
| **razorpayPaymentLinkId** | When helper submits withdraw; returned by `POST /v1/payment_links` | Firestore `payoutRequests` doc |
| **paymentLinkUrl** | Same as above (short_url from Razorpay) | Firestore `payoutRequests` doc |
| **razorpayPaymentId** | When the **payment link is paid** (callback URL or webhook from Razorpay) | Firestore `payoutRequests` doc (your backend updates it) |
| **razorpayPayoutId** | When **admin releases** payment via **Payouts API** (`POST /v1/payouts`) | Firestore `payoutRequests` doc (your backend updates it) |
| **status** | `pending` at creation; admin sets `processed` or `rejected` after release/failure | Firestore `payoutRequests` doc |

So:

- **Payment link** is created at **withdraw time** and its ID/URL are stored on the payout request.
- **razorpayPaymentId** is set when **the link is paid** (optional; only if you use the link to collect payment).
- **razorpayPayoutId** and **status** are set when **admin approves and releases** payment via the Payouts API.

---

## 3. Where to see it in Razorpay (with direct links)

| What | Where in Razorpay | Direct link |
|------|-------------------|-------------|
| **Payment link** (created when user taps **Submit Payout Request**) | Dashboard → **Payment Links** | **[https://dashboard.razorpay.com/app/paymentlinks](https://dashboard.razorpay.com/app/paymentlinks)** — list of payment links with amount, reference_id, status (Created/Paid/etc.). |
| **Payment** (when someone pays the link) | Dashboard → **Payments** | **https://dashboard.razorpay.com/app/payments** — search by payment ID (`pay_xxx`). |
| **Money sent to user (withdraw done)** – list of payouts, each with status | **RazorpayX → Payouts** | **RazorpayX (Banking) dashboard:** Log in at **[https://x.razorpay.com](https://x.razorpay.com)** then open **Payouts** in the menu. Or from main dashboard: **[https://dashboard.razorpay.com](https://dashboard.razorpay.com)** → **Banking** / **RazorpayX** → **Payouts**. Here you see the **list of payouts** to bank accounts; each row has **status** (e.g. processing, processed, failed, reversed). |

**Payouts – direct access:**  
- RazorpayX login: **[https://x.razorpay.com](https://x.razorpay.com)** — after login, click **Payouts** in the left/side menu to see the list of payouts and their status.  
- If your Razorpay dashboard uses a single domain, go to **[https://dashboard.razorpay.com](https://dashboard.razorpay.com)** and open **Banking** or **RazorpayX**, then **Payouts**.

So: **Payment Links** → [dashboard.razorpay.com/app/paymentlinks](https://dashboard.razorpay.com/app/paymentlinks). **Payouts (money sent to user, status)** → **x.razorpay.com** (Payouts) or **dashboard.razorpay.com** → Banking → Payouts.

---

## 4. Admin approve → payment release (how to do it properly)

When admin approves a payout request:

1. **Get the payout request** from Firestore (`payoutRequests` where `status == 'pending'`). You need: `helperId`, `amount`, bank details (or helper’s `razorpayFundAccountId` if already created).
2. **Create fund account** for the helper (if not already): use Razorpay Contacts + Fund Accounts API with bank details from the request. Get `fund_account_id`.
3. **Call Razorpay Payouts API** (`POST /v1/payouts`) with:
   - `fund_account_id`
   - `amount` in paise
   - `reference_id` = e.g. `payout_<requestId>` (so you can match it in Razorpay)
4. Razorpay returns **payout id** (e.g. `payout_xxx`). This is **razorpayPayoutId**.
5. **Update Firestore:**
   - Update the **payout request** doc: set `status: 'processed'`, `razorpayPayoutId`, `processedAt: now`.
   - **Reduce** the user’s **pendingPayout** by the request amount (so the app shows correct “Pending payout” and the user sees the request as completed).

In this app you can use:

- **FirestoreService.updatePayoutRequestByRequestId(** `requestId`, **status:** `'processed'`, **razorpayPayoutId:** `payoutId`, **processedAt:** now, **helperId:** `helperId`, **amount:** `amount` **)**  
  so that the payout request is updated and the user’s **pendingPayout** is decreased in one place.

If payout fails (e.g. Razorpay returns error), set `status: 'rejected'` and optionally `rejectionReason`; do **not** reduce pendingPayout (so the user can retry or you can fix and reprocess).

---

## 5. razorpayPaymentId (when the payment link is paid)

**Payment Links** are for **collecting** payment (someone opens the link and pays). So:

- If you **don’t** use the link for payment (e.g. you only use Payouts to send money from your balance to the helper), you **don’t** need to set **razorpayPaymentId** on the payout request. You only need **razorpayPayoutId** when admin releases.
- If you **do** use the link (e.g. admin or someone pays the link to “fund” the payout), then when Razorpay notifies you (callback or webhook) that the link was paid, you get **razorpay_payment_id**. Your backend should then update the payout request document in Firestore and set **razorpayPaymentId** to that value. That way you have a clear link between the payout request and the payment in Razorpay.

So: **razorpayPaymentId** is set on **your side** (backend) when you receive the payment success for the payment link (callback or webhook). It is **not** set by the app at withdraw time; at withdraw time we only set **razorpayPaymentLinkId** and **paymentLinkUrl**.

---

## 6. Link created – next steps: how the user gets money and where to check

Once the payment link is **created** and you see it in Razorpay at **Payment Links** (e.g. status **Created**, with Payment Link Id, Amount, Reference Id, Customer, Payment Link URL):

### How the user (helper) actually gets money

- **Payment Links** are for **receiving** payment (someone opens the link and pays → money comes to your Razorpay account).
- **Withdraw** means sending money **to** the helper’s bank. That is done via **Payouts**, not by the helper “paying” the link.
- So the **next step** is: **you (admin/backend) send money to the helper’s bank** using **Razorpay Payouts** (RazorpayX → Payouts, or Payouts API). That is when the user “gets” the money.

### Where to check in the Razorpay dashboard

| What | Where to check | What you see |
|------|----------------|--------------|
| **Payment link created** | **Dashboard → Payment Links** ([dashboard.razorpay.com/app/paymentlinks](https://dashboard.razorpay.com/app/paymentlinks)) | Your link: Payment Link Id, Amount, Reference Id, Customer, Payment Link URL, **Status** (e.g. Created, Paid, Expired). Use **Reference Id** to match with your Firestore payout request. |
| **Payment (if someone paid the link)** | **Dashboard → Payments** | Search by payment ID; status (Captured, etc.). Only relevant if you use the link to collect payment. |
| **Payout – money sent to user (withdraw done)** | **RazorpayX → Payouts** | **Link:** Log in at **[https://x.razorpay.com](https://x.razorpay.com)** then click **Payouts** in the menu. Or **dashboard.razorpay.com** → **Banking** / **RazorpayX** → **Payouts**. You see the **list of payouts** to bank accounts; each has **status** (processing / processed / failed). |

### Next steps (in order)

1. **Link created** ✓ – You see it under Payment Links (status e.g. **Created**).
2. **Admin approves the payout request** – From your app/backend, get the pending payout request (Firestore `payoutRequests` where `status == 'pending'`). You need the helper’s bank details (or fund account id).
3. **Create a Payout** – In Razorpay: go to **RazorpayX → Payouts** and create a payout to the helper’s bank (or call `POST /v1/payouts` with fund_account_id and amount). This sends the money to the user’s account.
4. **Check payout status** – In **RazorpayX → Payouts** you’ll see the payout status (e.g. processing → processed when the bank credit is done).
5. **Update your app** – When the payout is successful, update the payout request in Firestore: set **status: 'processed'**, **razorpayPayoutId** (from the payout you created), **processedAt**, and reduce the user’s **pendingPayout**. Then the user sees the withdraw as complete and the money is in their bank.

So: **Payment Links** = link created and visible in dashboard; **Payouts** = where you actually pay/withdraw to the user’s account and where you check “money sent” status. The **next step** after the link is created is to do the **Payout** (and then update Firestore) so the user gets the money and the status is “done”.

---

## 7. Summary

- **Submit Payout Request (button tap):** App creates a **Razorpay Payment Link** via `POST /v1/payment_links`, stores **razorpayPaymentLinkId** and **paymentLinkUrl** on the payout request, deducts coins and increases pendingPayout.
- **Where the link shows:** **https://dashboard.razorpay.com/app/paymentlinks** (Razorpay Dashboard → Payment Links). The link for that withdraw request appears there with amount and reference_id.
- **Payment release:** When admin approves, backend calls **Payouts API** to send money to the helper’s bank, then **sets** the payout request: **status: 'processed'**, **razorpayPayoutId**, **processedAt**, and reduces the user’s **pendingPayout**. That is where and how **payment release** is set for this flow.

This keeps the flow consistent: **Submit Payout Request** → link created → link visible at [Payment Links](https://dashboard.razorpay.com/app/paymentlinks) → admin does payout → payment release set on the payout request (status + razorpayPayoutId).
