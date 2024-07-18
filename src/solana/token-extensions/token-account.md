### Token Accounts
An account in Solana is an contiguous area of memory occupied by some bytes where these bytes represent state.
In this article we will look at Token Accounts as specified in the Token Extensions `(Token 2022)`.

Token accounts are used to store the information about a token. It has the following 8 fields.
> 1. `Mint Public Key` - This is the Public Key of the Mint associated with this account
> 2. `Owner Public Key` - Represents the Public Key of the Owner of the account
> 3. `Amount` - The amount of tokens
> 4. `Optional Delegate Public Key` - If there is a Public Key authorized to manage a delegated amount
> 5. `Account State` - Whether an account is not initialized, is initialized or if the account is frozen
> 6. `Optional Is Native` - Shows the rent-exempt amount if this is a native token if the value is provided
> 7. `Delegated Amount` - The amount the Delegate Public Key can manage
> 8. `Optional Close Authority` - The public key of authorized to close the account if it is provided.
>

#### Let's write some Rust code to implement the structure of the token account. This tutorial assumes you have installed the Rust stable toolchain `(version >= 1.78.0 at time of writing this)`

- Create a new rust `cargo project`
    ```sh
    $ cargo new token-account-layout 

    # Switch to the project directory
    $ cd token-account-layout
    ```
- Inside the `Cargo.toml` file, we then import `rand` crate that will generate random 32 bytes to simulate a Public Key since onchain there is no random number generation capability.
    ```TOML
    [package]
    name = "token-account-layout"
    version = "0.1.0"
    edition = "2021"

    [dependencies]
    rand = "0.8.5" #<-- Add this dependency
    ```
 **In the `main.rs` file**

***Note that Solana uses memory alignment to improve performance since modern CPUs perform better if they are reading or writing to aligned memory. You can do your own research about memory alignment.***

In an optional value of the Token Account, due to memory alignment Solana will represent the whether a field is optional using 4 bytes:
```
where:
    The value exists `Option::Some`  = [1u8, 0, 0, 0]
    The value doesn't exist `Option::None` = [0u8;4]
```

#### Add the rust code to represent a padded optional value.
```rust
/// If the value is Option::Some(value)
const OPTION_IS_SOME: [u8; 4] = [1u8, 0, 0, 0];
```

#### The Account Size is of  `165 bytes` in length. Define this constant in the code
```rust
/// The length of the contiguous chunk of memory representing a token account.
const TOKEN_2022_ACCOUNT_LEN: usize = 165;
```
##### An image representing the account structure
<image src="../../images/solana-token-account-size.png" width="80%">

`Source: Solana Cookbook`

#### Each byte range corresponds to a particular field in a Token account. Let's define these ranges as constants
```rust
/// Import `RangeInclusive` from rust core library
use core::ops::RangeInclusive;

/// Mint's Ed25519 Public key of 32 bytes in length
const MINT_ACC_RANGE: RangeInclusive<usize> = 0..=31;

/// Owner's Ed25519 Public key of 32 bytes in length
const MINT_OWNER_RANGE: RangeInclusive<usize> = 32..=63;

/// Eight bytes that hold an amount of type u64
const AMOUNT_RANGE: RangeInclusive<usize> = 64..=71;

/// 4 bytes of a Option + Ed25519 Public key of 32 bytes in length
const DELEGATE_RANGE: RangeInclusive<usize> = 72..=107;

/// One byte representing the Account state
const STATE_RANGE: usize = 108;

/// 4 Bytes Option + 8 bytes amount of type u64
/// representing current amount of rent exempt
// amount of a native token
const IS_NATIVE_RANGE: RangeInclusive<usize> = 109..=120;

/// 8 bytes of an amount of type u64 the delegate
/// can manage
const DELEGATED_AMOUNT_RANGE: RangeInclusive<usize> = 121..=128;

/// 4 bytes of a Option Type representation + Ed25519 Public key of 32 bytes in length
const CLOSE_AUTHORITY_RANGE: RangeInclusive<usize> = 129..=164;
```


#### Next, we define a struct to hold the Ed25519 public key (not that Solana public keys can be on the Ed25519 curve or off the curve)
```rust
/// Derive `Clone`, `Copy` to allow copying and cloning the Struct.
/// Derive `Default` to allow definition of a zero filled 32byte array.
/// Derive `PartialEq` to allow comparing of two structs
#[derive(Clone, Copy, Debug, Default, PartialEq)]
struct Ed25519PublicKey([u8; 32]);

impl Ed25519PublicKey {
    // generate 32 bytes random numbers
    fn new_random() -> Self {
        use rand::{thread_rng, Rng};

        let mut buffer = [0u8; 32];
        thread_rng().fill(&mut buffer[..]);

        Self(buffer)
    }

    /// Convert to 32 bytes
    fn to_bytes(&self) -> [u8; 32] {
        self.0
    }
}

impl From<&[u8]> for Ed25519PublicKey {
    fn from(bytes: &[u8]) -> Self {
        let valid_bytes: [u8; 32] = bytes
            .try_into()
            .expect("Invalid slice length for Ed25519 public key. 32 bytes are required");

        Self(valid_bytes)
    }
}
```

#### In Solana, a token account state can be `Uninitialized`, `Initialized` or `Frozen`.
```rust
/// The state of a token account
#[derive(Clone, Copy, Debug, Default, PartialEq)]
enum Token2022AccountState {
    /// not yet initialized
    #[default]
    Uninitialized = 0,
    /// is initialized allowing actions by the owner or delegate
    Initialized = 1,
    /// Account is frozen by the mint freeze authority preventing
    /// Any actions from the account owner or delegate
    Frozen = 2,
}

/// The `Token2022AccountState` is converted to
/// a byte representation by the numbers described
/// in its fields, like `Uninitialized = 0` 
/// using `Token2022AccountState::Uninitialized as u8`
/// converts that field to `0`
impl From<u8> for Token2022AccountState {
    fn from(value: u8) -> Self {
        match value {
            0 => Self::Uninitialized,
            1 => Self::Initialized,
            2 => Self::Frozen,
            _ => panic!("The byte value passed is not supported"),
        }
    }
}
```

#### Construct the Token Account struct
```rust

#[derive(Clone, Copy, Debug, Default, PartialEq)]
struct Token2022Account {
    /// The mint associated with this account
    mint: Ed25519PublicKey,
    /// The owner of this account.
    owner: Ed25519PublicKey,
    /// The amount of tokens this account holds.
    amount: u64,
    /// If `delegate` is `Some` then `delegated_amount` represents
    /// the amount authorized by the delegate
    delegate: Option<Ed25519PublicKey>,
    /// The account's state
    state: Token2022AccountState,
    /// If is_some, this is a native token, and the value logs the rent-exempt reserve. An Account
    /// is required to be rent-exempt, so the value is used by the Processor to ensure that wrapped
    /// SOL accounts do not drop below this threshold.
    is_native: Option<u64>,
    /// The amount delegated
    delegated_amount: u64,
    /// Optional authority to close the account.
    close_authority: Option<Ed25519PublicKey>,
}
```

#### Create helper methods to reuse code
```rust
impl Token2022Account {
    /// Converts bytes to an amount
    fn to_amount(bytes: &[u8]) -> u64 {
        // Little-endian is used
        u64::from_le_bytes(
            bytes
                .try_into()
                .expect("Invalid slice length for amount. 8 bytes are required"),
        )
    }

    /// Converts the bytes of an optional public key to Option<Ed25519PublicKey>
    fn to_optional_pubkey(bytes: &[u8]) -> Option<Ed25519PublicKey> {
        // if the first byte is zero then the public key is None
        if bytes[0] == 0 {
            Option::None
        } else {
            // Skip the first 4 bytes.
            // Since we implemented `From<[u8]> for Ed25519PublicKey`
            // using the `into()` method here auto-converts the bytes to
            // Ed25519PublicKey 
            Option::Some(bytes[4..].into())
        }
    }

    /// Converts an optional amount to a `Option<u64>`
    fn to_optional_amount(bytes: &[u8]) -> Option<u64> {
        // if the first byte is 0 then the value is None
        if bytes[0] == 0 {
            Option::None
        } else {
            // skip the first 4 bytes
            Option::Some(Self::to_amount(&bytes[4..]))
        }
    }
}
```

#### With the `impl` we create methods to pack `Self` and unpack bytes to `Self`
```rust
impl {
    // ... previous code here

    /// An optional public key consists of 36 bytes where
    /// 32 bytes public key and a byte representing whether
    /// it is Option::Some which is represented by `1` or
    /// if it is Option::None which is represented by `0`.
    /// However due to memory alignment, 3 bytes padding is added 
    /// (1 byte optional representation + 3 bytes padding)
    fn pack_optional_pubkey(option_pubkey: Option<&Ed25519PublicKey>) -> [u8; 36] {
        let mut buffer = [0u8; 36];

        if let Some(pubkey) = option_pubkey {
            // The byte range of the Option representation with 3 bytes padding
            buffer[0..=3].copy_from_slice(&OPTION_IS_SOME);
            // Add the 32 bytes for the public key after the padding
            buffer[4..=35].copy_from_slice(&pubkey.to_bytes());
        }

        buffer
    }


    /// Pack the Token2022Account struct (Self)
    fn pack(&self) -> [u8; 165] {
        // Initialize an empty buffer with 165 bytes of zeroes
        let mut buffer = [0u8; TOKEN_2022_ACCOUNT_LEN];

        // The range within the buffer for the `mint public key`
        buffer[MINT_ACC_RANGE].copy_from_slice(&self.mint.to_bytes());

        // The range within the buffer for the `owner public key`
        buffer[MINT_OWNER_RANGE].copy_from_slice(&self.owner.to_bytes());

        // The range within the buffer for the `8 bytes amount`
        buffer[AMOUNT_RANGE].copy_from_slice(&self.amount.to_le_bytes());

        // The range within the buffer for the `optional delegate public key`
        buffer[DELEGATE_RANGE].copy_from_slice(&Self::pack_optional_pubkey(self.delegate.as_ref()));

        // The range within the buffer for the `token account state`
        buffer[STATE_RANGE] = self.state as u8;

        // The range within the buffer for the `is_native optional amount`
        if let Some(amount) = self.is_native.as_ref() {
            // Define a buffer with 1 byte Option representation
            // 3 bytes padding for the Option representation
            // and 8 bytes amount
            let mut inner_buffer = [0u8; 12];
            // Load the optional bytes with the amount
            inner_buffer[0..=3].clone_from_slice(&OPTION_IS_SOME);
            // Add the 8 bytes amount
            inner_buffer[4..].copy_from_slice(&amount.to_le_bytes());
            // Copy this buffer into the buffer for the packed amount
            // within the range
            buffer[IS_NATIVE_RANGE].copy_from_slice(&inner_buffer)
        }

        // The range within the buffer for the `8 bytes delegate amount`
        // which the delegate authority can manage
        buffer[DELEGATED_AMOUNT_RANGE].copy_from_slice(&self.delegated_amount.to_le_bytes());

        // The range within the buffer for the `optional close authority public key`
        buffer[CLOSE_AUTHORITY_RANGE]
            .copy_from_slice(&Self::pack_optional_pubkey(self.close_authority.as_ref()));

        buffer
    }

    /// Code to unpack 165  bytes to `Token2022Account`
    fn unpack_account_data(packed: [u8; TOKEN_2022_ACCOUNT_LEN]) -> Self {
        // Since we derived default for `Token2022Account` we can then
        // we instantiate a `Token2022Account` with default values
        let mut account = Token2022Account::default();

        // Since `From<u8> for Ed25519PublicKey` is implemented,
        // use `.into()` method to auto-convert the byte range to an Ed25519PublicKey
        account.mint = packed[MINT_ACC_RANGE].into();

        // Same as above
        account.owner = packed[MINT_OWNER_RANGE].into();
        
        // Parse a byte slice to u64
        account.amount = Self::to_amount(&packed[AMOUNT_RANGE]);

        // Parse a byte slice to an optional public key
        account.delegate = Self::to_optional_pubkey(&packed[DELEGATE_RANGE]);

        // Parse a byte into the `AccountState` using `.into()` method
        // since `From<u8>` is implemented for `Token2022AccountState`
        account.state = packed[STATE_RANGE].into();

        // Parse a byte slice to an optional u64 amount
        account.is_native = Self::to_optional_amount(&packed[IS_NATIVE_RANGE]);
        
        // Parse a byte slice into optional u64 amount
        account.delegated_amount = Self::to_amount(&packed[DELEGATED_AMOUNT_RANGE]);

        // Parse a byte slice inot an optional public key
        account.close_authority = Self::to_optional_pubkey(&packed[CLOSE_AUTHORITY_RANGE]);

        account
    }
} 
```

#### Lastly test the code by comparing the packed bytes to the original.
```rust
fn main() {
    let account = Token2022Account {
        mint: Ed25519PublicKey::new_random(),
        owner: Ed25519PublicKey::new_random(),
        amount: 4,
        delegate: Option::Some(Ed25519PublicKey::new_random()),
        state: Token2022AccountState::Frozen,
        // If is_some, this is a native token, and the value logs the rent-exempt reserve. An Account
        // is required to be rent-exempt, so the value is used by the Processor to ensure that wrapped
        // SOL accounts do not drop below this threshold.
        is_native: Option::Some(2),
        delegated_amount: 1,
        close_authority: Option::Some(Ed25519PublicKey::new_random()),
    };

    let packed = account.pack();
    let unpacked = Token2022Account::unpack_account_data(packed);
    assert_eq!(&account, &unpacked);
}
```

That's it! A custom implementation of packing and unpacking a Token Account based on Solana's memory layout.

> CAUTION: This code is not production grade or optimized, it is meant to illustrate the byte ranges of a region of memory containing a Solana Token Account