# MultiversX DApp Architecture

## Frontend Layer
- **Authentication**: xPortal App, Ledger, DeFi Wallet, xAlias, Web Wallet, Passkey Proxy, Metamask Proxy.
- **SDK**: `sdk-dapp` (React/Typescript) handles connection and state.
- **Interaction**: Calls smart contract endpoints (ping, pong).

## Backend Layer (Blockchain)
- **Smart Contract**: Acts as the API.
- **Storage**: Mappers (`getAcceptedPaymentToken`, `getPingAmount`).
- **Events**: Signals successful operations (`pongEvent`).
- **Views**: Read-only data (`didUserPing`, `getTimeToPong`).

## Data Flow
1. User connects wallet (Frontend).
2. User initiates action (e.g., Ping).
3. Frontend creates transaction.
4. User signs transaction.
5. Transaction broadcasted to MultiversX network.
6. SC executes logic, updates storage, emits events.
7. Frontend updates state based on events/new queries.
