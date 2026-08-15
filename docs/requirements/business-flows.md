# Business Flows

### Normal Shopping & Purchase Journey

```
Search

↓

Search Suggestions (Auto Complete)

↓

Search Results

↓

Filter

↓

Sort

↓

Product Detail

↓

Select Variant

↓

Quantity

↓

Add To Cart

↓

Cart

↓

Checkout

↓

Login / Register / Guest

↓

Address

↓

Shipping

↓

Coupon

↓

Payment

↓

Payment Gateway

↓

Fraud Check

↓

Payment Success

↓

Order Creation

↓

Inventory Reserved

↓

Warehouse

↓

Packing

↓

Courier

↓

Shipment

↓

Delivered

↓

Review / Return / Refund
```

---

### User Registration, Login & Authentication Journey

```
User Opens App

↓

Sign Up / Login

↓

Enter Email or Phone

↓

OTP / Password Verification

↓

Social Login (Google / Apple / Facebook)

↓

Forgot Password

↓

Reset via OTP / Email Link

↓

Account Created

↓

Profile Setup

↓

Save Address / Payment Methods

↓

Session Token Issued

↓

Access Personalized Home Page
```

---

### Product Catalog Management (Admin)

```
Admin Login

↓

Admin Dashboard

↓

Product Management

↓

Add New Product

↓

Enter Product Info (Title, Brand, Description)

↓

Select Category

↓

Add Attributes (Category Schema)

↓

Upload Images / Video / 360 View

↓

Set Price

↓

Set Variants (Color, Size, Storage)

↓

Set Inventory / Stock

↓

Set SEO Meta Data

↓

Submit for Approval

↓

Product Review (QC Team)

↓

Approve / Reject

↓

Product Published

↓

Product Live on Storefront
```

---

### Category & Attribute Management (Admin)

```
Admin Dashboard

↓

Category Management

↓

Add New Category

↓

Set Parent / Child Category

↓

Define Category Schema (Attributes)

↓

Set Category Image / Banner

↓

Set Category Ranking / Display Order

↓

Save Category

↓

Category Published

↓

Category Visible in Navigation
```

---

### Inventory & Stock Management

```
Inventory Dashboard

↓

View Stock by Warehouse

↓

Update Stock Quantity

↓

Set Low Stock Threshold

↓

Set Reorder Alert

↓

Stock Sync Across Channels

↓

Reserve Stock (Order Placed)

↓

Deduct Stock (Order Confirmed)

↓

Release Stock (Order Cancelled / Expired)

↓

Stock Audit / Reconciliation
```

---

### Payment Processing & Fraud Check Journey

```
Checkout Initiated

↓

Select Payment Method

↓

Idempotency Key Generated

↓

Payment Gateway Request

↓

Fraud Detection Check

↓

3DS Authentication

↓

Payment Authorization

↓

Payment Capture

↓

Payment Success / Failure

↓

Payment Event Published

↓

Order Service Notified

↓

Refund (if applicable)

↓

Settlement / Reconciliation
```

---

### Order Management (Admin)

```
Admin Dashboard

↓

Order Management

↓

View Orders (New, Processing, Shipped, Delivered, Cancelled)

↓

Search / Filter Orders

↓

View Order Details

↓

Update Order Status

↓

Cancel Order

↓

Modify Order (Address / Item)

↓

Generate Invoice

↓

Assign to Warehouse

↓

Trigger Shipment

↓

Handle Order Escalation
```

---

### Shipping, Warehouse & Fulfillment Journey

```
Order Confirmed

↓

Inventory Reserved

↓

Warehouse Assignment

↓

Pick List Generated

↓

Picking

↓

Packing

↓

Quality Check

↓

Shipping Label Generated

↓

Courier Assignment

↓

Shipment Created

↓

Tracking Number Generated

↓

Packed

↓

Shipped

↓

In Transit

↓

Out For Delivery

↓

Delivered
```

---

### Returns, Refunds & Exchange Journey

```
Delivered Order

↓

Customer Requests Return / Exchange

↓

Select Reason

↓

Return Eligibility Check

↓

Approve / Reject Return

↓

Schedule Pickup

↓

Courier Pickup

↓

Warehouse Receives Item

↓

Quality Check

↓

Approve Refund / Exchange

↓

Refund to Original Payment / Wallet

↓

Exchange Item Shipped

↓

Customer Notified
```

---

### Notifications Journey (Email / SMS / Push)

```
Event Triggered (Order, Payment, Shipment, Offer)

↓

Notification Service

↓

Select Channel (Email / SMS / Push / WhatsApp)

↓

Template Rendering

↓

Personalization

↓

Send Notification

↓

Delivery Status Tracked

↓

Retry on Failure
```

---

### Reviews & Ratings Journey

```
Delivered Order

↓

Request for Review (Email / Push)

↓

Customer Writes Review

↓

Rate Product

↓

Upload Photos / Video

↓

Review Moderation

↓

Approve / Reject Review

↓

Review Published

↓

Seller Response

↓

Review Impacts Ranking
```

---

### Offers, Coupons & Promotions Management (Admin)

```
Admin Dashboard

↓

Promotion Management

↓

Create Coupon / Offer

↓

Set Discount Type (Flat / Percentage)

↓

Set Eligibility (Category / User / Min Order)

↓

Set Validity Period

↓

Set Usage Limit

↓

Publish Offer

↓

Offer Applied at Checkout

↓

Track Offer Usage

↓

Expire / Deactivate Offer
```

---

### Wishlist & Saved Items Journey

```
Product Detail Page

↓

Add to Wishlist

↓

Wishlist Page

↓

Price Drop Notification

↓

Move to Cart

↓

Remove from Wishlist

↓

Share Wishlist
```

---

### Customer Support & Order Tracking Journey

```
Customer Opens Support

↓

Select Issue Type

↓

Order Related / Product Related / Payment Related

↓

Chatbot / Self-Service

↓

Escalate to Live Agent

↓

Agent Reviews Order History

↓

Resolution (Refund / Replacement / Info)

↓

Ticket Closed

↓

Customer Feedback
```

---

### Roles & Permission Management (Admin)

```
Admin Dashboard

↓

User & Role Management

↓

Create Role (Admin, Manager, Support, Seller)

↓

Assign Permissions (Module Level Access)

↓

Assign Role to Staff User

↓

Update / Revoke Permission

↓

Audit Log of Access Changes
```

---

### Seller / Vendor Onboarding & Management

```
Seller Registration

↓

KYC Document Upload

↓

Business Verification

↓

Bank Account Verification

↓

Seller Approval

↓

Seller Dashboard Access

↓

Product Listing by Seller

↓

Commission / Fee Setup

↓

Seller Order Management

↓

Seller Payout / Settlement

↓

Seller Performance Rating
```

---

### Flash Sale / High Concurrency Purchase Journey

```
Sale Starts

↓

Millions of Users Refresh

↓

CDN Serves Static Assets

↓

Virtual Waiting Room

↓

Rate Limiter

↓

User Admitted

↓

Inventory Availability Checked

↓

Reserve Stock (Temporary Hold)

↓

Checkout Timer Starts

↓

Payment Authorization

↓

Idempotency Key Prevents Duplicate Payments

↓

Payment Succeeds

↓

Confirm Inventory Reservation

↓

Create Order

↓

Reduce Available Stock Atomically

↓

Release Reservation if Payment Fails / Times Out

↓

Send Confirmation

↓

Warehouse Fulfillment
```

---

### Analytics & Reporting Dashboard (Admin)

```
Admin Dashboard

↓

Analytics Module

↓

Sales Reports

↓

Inventory Reports

↓

Customer Behavior Analytics

↓

Marketing Campaign Performance

↓

Seller Performance Reports

↓

Export / Schedule Reports

↓

Data Feeds to BI Tools
```
