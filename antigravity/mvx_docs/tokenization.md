# Tokenization & Dynamic ESDT

## Philosophy: "Tokenization of Everything"
MultiversX allows native representation of any asset via ESDT.
- **Fungible**: Currency, Points.
- **SFT (Semi-Fungible)**: Tickets, Game Items, Coupons.
- **NFT**: Unique Art, Identity, Real World Assets.

## Dynamic ESDT
Standard ESDTs can be made dynamic to allow metadata updates.
- **`canUpgrade` Property**: Must be set to `true` during issuance or via `ESDTRoleESDTControl`.
- **Metadata Update**: The token manager (with `ESDTRoleNFTUpdate`) can change attributes of SFTs/NFTs.
- **Use Case**: An RPG character (SFT) gains XP. The contract updates its *Attributes* field to reflect the new Level.

## SFT as Data Containers
SFTs are powerful storage mechanisms.
- **Attributes Field**: Can store serialized data (structs).
- **No Contract Storage**: Data lives in the user's wallet (in the token), reducing contract storage costs.
- **Composability**: SFTs can be transferred to contracts which "read" the attributes and act accordingly.

## Meta-ESDT
A special SFT acting as a "wrapped" version of a fungible token, enabling it to hold metadata/attributes while behaving like a fungible token in some contexts.
