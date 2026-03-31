# StockPilot IMS

Inventory management system built with React, Vite, Tailwind CSS, Firebase Auth, Cloud Firestore, and Firebase Hosting.

## Free-tier architecture

- Frontend: `web/` contains the React SPA.
- Data layer: product creation, stock adjustments, and soft delete now run directly from the client with Firestore `writeBatch` and `runTransaction`.
- Security: `firestore.rules` validates ownership, movement logging, quantity transitions, and soft delete rules.
- Hosting: `firebase.json` deploys Hosting plus Firestore rules/indexes only.

This keeps the app compatible with the Firebase free tier because it no longer depends on Cloud Functions.

## Firebase projects

The current aliases in `.firebaserc` are:

- `dev` -> `projeto-inventario-ads-dev`
- `prod` -> `projeto-inventario-ads`

## Firestore collections

### `users/{uid}`

```json
{
  "id": "uid",
  "email": "user@example.com",
  "displayName": "Alex Johnson",
  "createdAt": "Timestamp",
  "updatedAt": "Timestamp"
}
```

### `products/{productId}`

```json
{
  "id": "generated-id",
  "name": "Wireless scanner",
  "description": "Warehouse handheld scanner",
  "category": "Peripherals",
  "price": 149.9,
  "quantity": 12,
  "createdAt": "Timestamp",
  "updatedAt": "Timestamp",
  "userId": "uid",
  "deleted": false,
  "lastMovementId": "movement-id"
}
```

### `movements/{movementId}`

```json
{
  "id": "generated-id",
  "productId": "product-id",
  "userId": "uid",
  "type": "IN",
  "quantity": 4,
  "reason": "Purchase order #204",
  "createdAt": "Timestamp"
}
```

## Key flows

### Product creation

`web/src/services/productService.js` creates the product document and, when `initialQuantity > 0`, writes the initial `IN` movement in the same batch.

### Stock adjustment

`web/src/services/productService.js` uses a Firestore transaction to:

- load the current product
- validate ownership and stock floor
- update the quantity
- create the movement entry
- store `lastMovementId` on the product

### Firestore rule enforcement

`firestore.rules` allows:

- product metadata edits without changing quantity
- stock changes only when they are paired with a valid movement document
- soft delete only for the owner
- movement creation only when it matches the corresponding product mutation

## Local setup

1. Install dependencies:

```bash
npm install
```

2. Create the environment files:

```bash
cp web/.env.dev.example web/.env.dev
cp web/.env.prod.example web/.env.prod
```

3. Fill each file with the Firebase Web App config from the corresponding project.

4. Start the app:

```bash
npm run dev
```

## Firebase console setup

For each project, enable only:

- Authentication with Email/Password
- Cloud Firestore
- Firebase Hosting

You do not need Cloud Functions or a paid plan for this version.

## GitHub Actions secrets

The deploy jobs run inside the GitHub environments named `dev` and `prod`.
Store the matching secrets in those environments, or define them as repository
secrets if you prefer a shared configuration.

Required secrets for `dev`:

- `FIREBASE_SERVICE_ACCOUNT_DEV`
- `DEV_FIREBASE_API_KEY`
- `DEV_FIREBASE_AUTH_DOMAIN`
- `DEV_FIREBASE_PROJECT_ID`
- `DEV_FIREBASE_STORAGE_BUCKET`
- `DEV_FIREBASE_MESSAGING_SENDER_ID`
- `DEV_FIREBASE_APP_ID`

Required secrets for `prod`:

- `FIREBASE_SERVICE_ACCOUNT_PROD`
- `PROD_FIREBASE_API_KEY`
- `PROD_FIREBASE_AUTH_DOMAIN`
- `PROD_FIREBASE_PROJECT_ID`
- `PROD_FIREBASE_STORAGE_BUCKET`
- `PROD_FIREBASE_MESSAGING_SENDER_ID`
- `PROD_FIREBASE_APP_ID`

## Deployment

- `npm run deploy:dev` deploys `hosting`, `firestore:rules`, and `firestore:indexes` to alias `dev`
- `npm run deploy:prod` deploys the same targets to alias `prod`

GitHub workflows:

- `.github/workflows/deploy-dev.yml`
- `.github/workflows/deploy-prod.yml`

## Notes

- The `functions/` folder can remain in the repository as legacy reference code, but it is no longer deployed or installed as part of the active workspace.
- Existing products continue to work. New writes add `lastMovementId` automatically.
