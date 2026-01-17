# evm-portfolio-dapp

A MetaMask Portfolio dApp that fetches an EVM wallet's token balances and USD values using the Moralis EVM API. The project is split into a small Express backend that queries Moralis and a Next.js frontend that handles authentication (MetaMask → Moralis challenge → NextAuth) and displays the portfolio.

---

## Features
- Authenticate with MetaMask (signature-based auth via Moralis + NextAuth).
- Fetch token balances and USD prices using Moralis EVM API.
- Present portfolio allocation, price, and balance per token.
- Simple, modular React components (cards, table header/content).

## Tech stack
- Backend: Node.js (Express), Moralis SDK
- Frontend: Next.js, React, Wagmi (for wallet), @moralisweb3/next, next-auth, axios
- Communication: Frontend -> Backend over HTTP (axios)

---

## Architecture (high level)
Frontend (Next.js)
  - Wagmi + MetaMask -> request sign-in challenge via MoralisNextApi -> sign message -> next-auth
  - After auth, calls backend /gettokens to fetch token balances and prices
Backend (Express)
  - Uses Moralis SDK to call Moralis.EvmApi.token.getWalletTokenBalances & getTokenPrice
  - Returns token list + total USD value

ASCII diagram:
Frontend (3000) --[auth challenge/signature]--> Moralis API (via /api/moralis proxy)
Frontend --[GET /gettokens?address=...]--> Backend (5001) --[Moralis SDK]--> Moralis EVM API

---

## Prerequisites
- Node.js (v16+ recommended)
- npm or yarn
- A Moralis account and an EVM API key (MORALIS_API_KEY)
- MetaMask browser extension (for local wallet authentication)

---

## Environment variables

The project uses environment variables in both backend and frontend. Create two separate files: one for the backend and one for the frontend.

Backend (.env in /backend)
MUST contain:
```
MORALIS_API_KEY=your_moralis_api_key_here
```

Frontend (.env.local in /frontend)
Recommended:
```
MORALIS_API_KEY=your_moralis_api_key_here
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=a_long_random_string_for_production
```

Notes:
- MORALIS_API_KEY is used by both backend and frontend (the frontend uses the MoralisNextApi helper).
- NEXTAUTH_URL should match your frontend host for NextAuth callbacks (http://localhost:3000 for local dev).
- NEXTAUTH_SECRET is strongly recommended in production for session signing.

---

## Install & Run

Quickstart (local development):

1. Clone the repo
```
git clone https://github.com/NovaDev5/evm-portfolio-dapp.git
cd evm-portfolio-dapp
```

2. Backend
```
cd backend
npm install
# create .env (see env section)
node index.js
# or
npm start
```
The backend listens by default on port 5001 and exposes GET /gettokens.

3. Frontend
In a new terminal:
```
cd frontend
# using npm
npm install
# or using yarn
# yarn
# create .env.local (see env section)
npm run dev
```
Frontend runs at http://localhost:3000 (Next's default).

Important: The frontend's getWalletTokens component calls http://localhost:5001/gettokens — ensure the backend is running on port 5001 or change the endpoint in `frontend/components/getWalletTokens.js`.

---

## Usage (local)
1. Visit http://localhost:3000
2. Click "Connect Metamask" (or Authenticate) — this triggers:
   - Wagmi connects to MetaMask and fetches your account
   - The frontend requests a challenge from the Moralis auth endpoint
   - You sign the message in MetaMask
   - The app calls next-auth signIn("moralis-auth", ...) to create a session
3. After successful login you'll be redirected to `/user` where the dashboard fetches tokens via the backend and displays them.

---

## API Reference

GET /gettokens
- Endpoint (backend): http://localhost:5001/gettokens
- Query parameters:
  - address (required) — the wallet address to query
- Response: an array where each element except the last is an object with token info and the last element is a number containing the total wallet USD value. Example shape (current implementation):

[
  {
    walletBalance: {
      token_address: "0x...",
      name: "Token",
      symbol: "TKN",
      decimals: 18,
      thumbnail: "https://...",
      balance: 1234560000000000000,
      ...
    },
    calculatedBalance: "1.23",        // human readable balance (string)
    usdPrice: 2.34                    // price per token in USD (number)
  },
  ...
  1234.56                             // total wallet USD value (number) — appended as last array element
]

Notes:
- The current frontend expects the total at index 3 (`tokens[3]`) which is brittle. See "Known issues & Improvements" below.

---

## Authentication Flow (brief)
- Frontend uses @moralisweb3/next's `useAuthRequestChallengeEvm` to request a challenge message for the connected MetaMask account.
- User signs the challenge message via Wagmi's signMessage.
- The signed message and raw challenge are passed to NextAuth via `signIn('moralis-auth' ...)`.
- NextAuth's MoralisNextAuthProvider validates the signature via Moralis services and issues a session stored in cookies.
- Server-side routes (getServerSideProps) can validate session to protect pages (see frontend/pages/user.jsx).

---

## Frontend component overview
- pages/index.jsx: Landing page with "Connect Metamask" button to start auth.
- pages/signin.jsx: Alternate page with similar auth flow.
- pages/user.jsx: Protected dashboard that requires a NextAuth session.
- components/getWalletTokens.js: axios GET to backend /gettokens to fetch balances (calls localhost:5001).
- components/card.js: Displays a token's thumbnail, symbol, name, portfolio %, USD price, and balance.
- components/tableHeader.js / tableContent.js / loggedIn.js: UI layout for token list.

---

## Known issues & recommended fixes
1. Response shape (total is appended to tokens array)
   - Current backend pushes total wallet USD as the last element of the token array. The frontend then expects a fixed index (tokens[3]) which is brittle and breaks when token count != 3.
   - Recommended: return an object with a tokens array and totalUsd field:

Suggested backend change (example):
```js
// In backend/index.js, instead of modifiedResponse.push(totalWalletUsdValue);
const result = {
  tokens: modifiedResponse,
  totalUsdValue: totalWalletUsdValue
};
return res.status(200).json(result);
```

And update frontend GetWalletTokens:
```js
// setTokens(response.data.tokens);
setTokens(response.data.tokens);
setTotalUsd(response.data.totalUsdValue);
```

2. Hardcoded chain
   - Backend currently requests balances on chain "0x1" (Ethereum Mainnet). If you want multiple chains or dynamic chain selection, expose a `chain` query parameter and validate it.

3. CORS / endpoint mismatch
   - Frontend calls http://localhost:5001 directly. When deploying, configure a proper API base URL and enable HTTPS and CORS accordingly. Prefer making backend relative (e.g., via a proxy or through the same domain) in production.

---

## Troubleshooting
- "Address is undefined" in backend: Ensure Wagmi connection succeeded and `address` query param is being passed from frontend. The `getWalletTokens` component uses useAccount(); ensure it runs after wallet connection.
- CORS errors: Backend uses cors() but if you changed origins, set proper CORS options. Confirm ports and hostnames match (localhost vs 127.0.0.1).
- Auth failures (NextAuth callbacks): Ensure NEXTAUTH_URL is set correctly and NEXTAUTH_SECRET is set for production.
- Invalid Moralis API key: You will receive Moralis SDK errors; confirm MORALIS_API_KEY value is correct and not expired.

---

## Security & Production Notes
- Do NOT commit your MORALIS_API_KEY, NEXTAUTH_SECRET, or any secrets to source control.
- Consider proxying Moralis calls server-side only and avoid exposing API keys to client-side code. In the current setup, the frontend uses MoralisNextApi which reads MORALIS_API_KEY server-side, but double-check that no secrets are leaked.
- Use HTTPS and secure cookie flags when deploying (NextAuth settings & hosting).
- Rate-limiting and caching: Moralis token price calls may be rate-limited — consider caching responses or aggregating calls.

---

## Potential improvements
- Return a structured JSON response from backend (tokens + total + metadata).
- Support dynamic chain selection (pass chain as query param).
- Add a small loading and error UI in GetWalletTokens component.
- Use environment-specific base URLs for the backend (so frontend doesn't call localhost in production).
- Add unit/integration tests for backend and critical frontend flows.
- Add Dockerfiles and docker-compose for local reproducible environments.

---

## Contributing
Contributions are welcome. Please open issues for bugs or feature requests. To contribute:
- Fork the repository
- Create a feature branch
- Open a pull request with a clear description of changes

---

## License
This repository currently does not include a license file. Add an appropriate LICENSE if you plan to share or use the code in other projects.

---

If you want, I can:
- Provide a patched backend.js that returns the structured JSON (tokens + totalUsdValue) and a matching frontend change in getWalletTokens.js.
- Create a Docker Compose example to run both services locally.
