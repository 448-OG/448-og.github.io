### Wallet Backup and Recovery
Source code — [https://github.com/448-OG/Btrust-Deliverables/tree/master/bip39-simple](https://github.com/448-OG/Btrust-Deliverables/tree/master/bip39-simple)

Seed phrases exist because private keys are big numbers and it’s hard for anyone to memorize them. For example the bytes below as hex.
> 3AF660000011144004F890FF

The hex characters above have repeating characters like `00000` and `4400` making it easy for someone to miss or add a character. When you’re dealing with digital wallets and cryptocurrencies, it’s really important to get everything right. If you make a mistake with your wallet’s private key, you could end up with an empty wallet. This is because even a small error can create a completely different key, leading to a different wallet.

This is where BIP39 comes in. It replaces complicated keys with a set of simple words, known as a mnemonic phrase. This makes it easier to manage your keys without making mistakes. It’s like turning a hard-to-remember password into a memorable sentence.

So, instead of dealing with complex and error-prone keys, you just need to remember your unique set of words. Several backup and recovery mechanisms have been proposed such as `BIP39`, `Electrum v2`, `Aezeed`, `Muun` and `SLIP39`.

In this article we will look at `Bitcoin Improvement Proposal 39 (BIP39)` and write some Rust code to create and recover a seed using a mnemonic.

`BIP39` is a technique that turns complex wallet recovery codes into a sequence of simple, readable words, known as a mnemonic. Here’s how it works:
1. The list of words used to create the mnemonic is designed so that typing just the first four letters of a word is enough to clearly identify it.
2.  The list avoids similar word pairs like `“build”` and `“built”`, or `“quick”` and `“quickly”`. These pairs can make the mnemonic harder to remember and more likely to be guessed incorrectly.
3. The words are sorted in a specific order. This makes it quicker and easier to find a particular word when you need it. It also allows for the use of a ‘trie’ (a type of search tree), which can help to compress the data.
4. The specification also requires that native characters used to be encoded in UTF-8 using     `Normalization Form Compatibility Decomposition (NFKD)`

Here are the steps to create a BIP39 mnemonic:
1. Create random bytes using a Cryptographically Secure Pseudo Random Number Generator. The bytes range between 128 bits (16bytes) to 256 bits (32 bytes) to generate 12–24 words.
2. Create a checksum, which is the first (length of entropy in bits/32) bits of the SHA256 of the entropy.
3. Append this checksum to the end of the initial entropy.
4. Split the result into groups of 11 bits.
5. Convert these bits into their decimal representation.
6. These decimal representations vary from 0–2047 and work as an index to a mnemonic word list. Choose words corresponding to these indices from the word list. For this tutorial we will use the most popular one which is English word list. Download the word list from [https://github.com/bitcoin/bips/blob/master/bip-0039/english.txt](https://github.com/bitcoin/bips/blob/master/bip-0039/english.txt)

Remember, the mnemonic generated from these steps can be used to recover your wallet, so it’s crucial to keep it safe and secure.

This tutorial assumes you have installed Rust Programming Language toolchain which comes bundled with cargo build and dependency management tool.

Let’s create a cargo project called bip39-simple
```sh
$ cargo new bip39-simple
```
Switch to that project directory
```sh
$ cd bip39-simple
```
Add dependencies to Cargo.toml file
```TOML
[dependencies]
rand_chacha = "*"
rand_core = { version = "*", features = ["getrandom"] }
sha2 = "*"
pbkdf2 = { version = "0.12.2", features = [
    "simple",
] }
```
We add the `getrandom` feature to `rand_core` in order to get random bytes using the operating system cryptographically secure random number generator. `sha2` crate will be used to create a checksum from the `SHA256` hash of the random bytes generated.

In our `main.rs` file we import these dependencies and bring some data types into scope
```rust
use pbkdf2::pbkdf2_hmac;
use rand_chacha::ChaCha20Rng;
use rand_core::{RngCore, SeedableRng};
use sha2::{Digest, Sha256, Sha512};
use std::{
    fs::File,
    io::{self, prelude::*},
    path::{Path, PathBuf},
};
```
The `PBKDF2` key derivation function requires that an iteration count and a salt. The `BIP39` standardized document requires that if a user includes a passphrase in the creation of a seed, that passphrase be appended to the word mnemonic and if there is no seed then we append an empty string `""` instead. The specification also requires an iteration count of `2048`. So we add two constants to hold these values.
```rust
/// Number of iterations to be run by the PBKDF2 for key derivation
pub const ITERATION_COUNT: u32 = 2048;
/// The word used as a prefix for the salt for our key derivation function
pub const SALT_PREFIX: &str = "mnemonic";
```
We need a way to generate random bytes so we introduce the a way to generate these bytes in a manner that a user can generate varying length of bytes (in our case between 128 to 256 bytes)
```rust
// This struct takes a constant `N` as a generic
// enabling one to specify a variable length for the bytes generated
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
pub struct Entropy<const N: usize>([u8; N]);

impl<const N: usize> Entropy<N> {
    // This method generates the bytes 
    pub fn generate() -> Self {
        // Instantiate our cryptographically secure random byte generation algorithm
        let mut rng = ChaCha20Rng::from_entropy();
        // Create a zero filled buffer to hold our bytes
        let mut buffer = [0u8; N];
        // Fill our buffer with random bytes
        rng.fill_bytes(&mut buffer);

        // Return our buffer
        Self(buffer)
    }
}
```
Next we need to create a struct that can hold methods to generate our mnemonic, create our seed and recover our seed.
```rust
#[derive(Debug, Default)]
pub struct Bip39Generator {
    // This holds all our indexes that we will use to fetch
    // our word from the word list 
    // with each index corresponding to an index
    // from our wordlist contained in a Vec<word>
    mnemonic_index: Vec<u16>,
    // This field holds the random bytes with our checksum
    // bytes appended to the end
    appended: Vec<u8>,
    // This contains a path to our wordlist file
    path: PathBuf,
}
```
Let’s create a method to instantiate our Bip39Generator struct.
```rust
impl Bip39Generator {
  // This method takes an argument `path_to_wordlist` which
  // is a path to the wordlist we downloaded
  // where the path is anything that implements the trait
  // AsRef<Path> meaning we pass any data type as convert it 
  // to a path using the `.as_ref()` method as long as that
  // data type implements the `AsRef<Path>` trait.
  pub fn new(path_to_wordlist: impl AsRef<Path>) -> Self {
      Self {
          // Convert `path_to_wordlist` argument to a path
          // using `.as_ref()` method and convert it
          // to a `std::path::PathBuf` using the `.to_path_buf()`
          path: path_to_wordlist.as_ref().to_path_buf(),
           // All other fields can hold default values
          // and we can call this method since 
          // we derived `Default` values using `#[derive(Default)]` 
          // on our struct 
          ..Default::default()
      }
  }
}
```
Next we create a method to load our word list from the file we download containing our English word list

Note the `...` is not part of Rust syntax but to show previous code we implemented.
```rust
impl Bip39Generator {
  ...

// This method takes in a mutable `Self`
fn load_wordlist(&mut self) -> io::Result<Vec<String>> {
        // open the file using the path we passed
        // when instantiating our struct 
        // using `Bip39Generator::new()`
        let file = File::open(&self.path)?;
        // Create a buffer so that we can efficiently readd
        // our file
        let reader: io::BufReader<File> = io::BufReader::new(file);

        // Create a Vector to hold our wordlist
        let mut wordlist = Vec::<String>::new();

        // Read each line
        for line in reader.lines() {
          // Push each word to our `wordlist` vector
          // handling any I/O errors using `?`
            wordlist.push(line?);
        }

        // Return our vector of word list
        Ok(wordlist)
    }
}
```
We need to generate a checksum to append to our randomly generated bytes in order to generate a mnemonic
```rust
impl Bip39Generator {
  ...
  // Here we pass our generated random bytes as `entropy` argument
   fn generate_checksum<const N: usize>(&mut self, entropy: [u8; N]) -> &mut Self {
      // BIP39 spec requires a seed to be generated
      // using a SHA256 Psuedo Random Function (PRF)
      // so we instantiate a SHA256 hashing function.
      let mut hasher = Sha256::new();

      // We now pass our random bytes into our SHA256 PRF
      hasher.update(entropy.as_slice());

      // We now get our finalized value. Using
      // SHA256 always ensures that despite being
      // able to use variable length of random bytes
      // we always get back a 256 bit (32 byte) value.
      let entropy_hash = hasher.finalize();

      // Since we get a 32 byte value we multiply by
      // `8` to get number of bits since 1 byte == 8 bits
      let bits_of_entropy = entropy.len() * 8;
      // We get our `n` bits for our checksum from the
      // length of the random bits (entropy) 
      // where `n` is calculated as the 
      // `length of our random bits / 32`
      let bits_of_checksum = bits_of_entropy / 32;
      // We then use bit shifting to get
      // bits of checksum from our
      // 256 bit hash in variable `entropy_hash`
      let significant = entropy_hash[0] >> bits_of_checksum;
  
      let mut appended = entropy.to_vec();
      // We then append our checksum to our random
      appended.push(significant);

      // We now assign our appended bytes to the `appended`
      // field of our `Bip39Generator` struct which is `Self`
      self.appended = appended;

      self
  }
}
```
The next step is to get the index of the words to be used in our mnemonic. This step involves splitting our `random bytes with the appended checksum into groups of 11 bits`.
```rust
impl Bip39Generator {
  ...

  // We pass a mutable to self since we want to
  // add the result of this computation to `Self`
  fn compute(&mut self) -> &mut Self {
    // This vector will hold the binary 
    // representation of each byte in the `appended` vector.
      let mut bits = vec![];

      // This line starts a loop that iterates over each byte in the `self.appended` vector.
      for &byte in self.appended.iter() {
          // This line starts a nested loop that 
          // counts backwards from 7 to 0. 
          // The variable `i` represents the position of 
          // the bit we're interested in within 
          // the current byte.
          for i in (0..8).rev() {
          /*
            This line does three things:
             - `byte >> i`: This is a right bitwise shift operation. 
                            It moves the bits in `byte` `i` places to the right. 
                            The bit at position `i` is now at position 0.
             - `(byte >> i) & 1u8`: This is a bitwise AND operation with `1u8` (which is `1` in binary). 
                                    This operation effectively masks all the bits in `byte` except for the one at position 0.
             - `bits.push((byte >> i) & 1u8 == 1);`: This pushes `true` if the bit at position 0 is `1` 
                                                      and `false` otherwise into the `bits` vector.
            */            
              bits.push((byte >> i) & 1u8 == 1);
          }
      }

      // This line starts a loop that iterates over 
      // the `bits` vector in chunks of 11 bits.
      for chunk in bits.chunks(11) {
          // This line checks if the current chunk has 
          // exactly 11 bits. If it does, the code inside 
          // the if statement is executed.
          if chunk.len() == 11 {
              // This line initializes a mutable 
              // variable named `value` and sets it to 0. 
              // This variable will hold the decimal 
              // representation of the current 11-bit chunk.
              let mut value: u16 = 0;
              
              // This line starts a nested loop that iterates
              // over each bit in the current chunk. 
              // The variable `i` is the index of the current
              //  bit, and `bit` is the value of the current bit.
              for (i, &bit) in chunk.iter().enumerate() {
                  // This line checks if the current bit 
                  // is `1` (true). If it is, it shifts `1` 
                  // to the left by `(10 - i)` places 
                  // (this effectively gives `1` a value of `2^(10 - i)`) 
                  // and then performs a bitwise OR operation with `value`. 
                  // This has the effect of adding `2^(10 - i)` to `value`.
                  if bit {
                      value |= 1u16 << (10 - i);
                  }
              }
              // This line pushes the decimal 
              // representation of the current 11-bit chunk 
              // into the `self.mnemonic_index` vector.
              self.mnemonic_index.push(value);
          }
      }

      self
  }
}
```
Now that we can get our mnemonic
```rust
impl Bip39Generator {
  ...
  
    pub fn mnemonic<const N: usize>(&mut self) -> io::Result<String> {
        // This generates the number of random bits we need
        let entropy = Entropy::<{ N }>::generate();

        // Next, let's generate our checksum
        self.generate_checksum::<N>(entropy.0);
  
        // Next we compute the decimal numbers we will use
        // to get our wordlist
        self.compute();

        // Load the wordlist into memory
        let wordlist = self.load_wordlist()?;

        // Iterate through the decimal numbers
        // and for each decimal number get the word
        // in it's index in the wordlist (wordlist[index from decimal number]
        let mnemonic = self
            .mnemonic_index
            .iter()
            // Enumerate to get the current count in our interation
            .enumerate()
            .map(|(index, line_number)| {
                // Convert our decimal index (line_numer) to 
                // a usize since Rust is very strict in that
                // you can only index an array using a usize
                // so we dereference and cast using `as usize`
                let word = (&wordlist[*line_number as usize]).clone() + " ";  // Add a space in each word
                // Since indexes start at zero we add `1`
                // to make them human readable (humans mostly count from 1)
                let index = index + 1;

                // Check if we have our index is less than
                // 10 so we add a padding to make printing
                // to console neat
                let indexed = if index < 10 {
                    String::new() + " " + index.to_string().as_str()
                } else {
                    index.to_string()
                };

                // Print our index and each word. This 
                // will show the user the words in each
                // line but with a number. eg
                //  9. foo
                // 10. bar
                println!("{}. {}", indexed, &word);

                // Return the word
                word
            })
            .collect::<String>(); // Combine all strings into one

        // Trim the last space in the and return the mnemonic
        Ok(mnemonic.trim().to_owned())
    }
}
```
Now we can generate a seed from our mnemonic
```rust
impl Bip39Generator {
...

  // We pass our mnemonic and an optional passphrase
    fn seed(mnemonic: &str, passphrase: Option<&str>) -> io::Result<Vec<u8>> {
        // We check if there is a passphrase provided.
        // if there is one we prefix our salt with the passphrase
        let salt = if let Some(passphrase_required) = passphrase {
            String::new() + SALT_PREFIX + passphrase_required
        } else {
            String::from(SALT_PREFIX)
        };

        // We want to generate a 512bit seed
        // so we create a buffer to hold this.
        let mut wallet_seed = [0u8; 64]; // 512 bits == 64 bytes

        // We generate a key and push all the bytes to the `wallet_seed` buffer
        pbkdf2_hmac::<Sha512>(
            mnemonic.as_bytes(),
            salt.as_bytes(),
            ITERATION_COUNT,
            &mut wallet_seed,
        );

        // We return our seed
        Ok(wallet_seed.to_vec())
    }
}
```
Create methods to get a seed secured with a passphrase and one without. These methods call `seed()` which is a private method.
```rust
impl Bip39Generator {
...
   /// Generates a seed without a passphrase
    pub fn insecure_seed(mnemonic: &str) -> io::Result<Vec<u8>> {
        Self::seed(mnemonic, None)
    }

    /// Generates a seed with a passphrase
    pub fn secure_seed(mnemonic: &str, passphrase: &str) -> io::Result<Vec<u8>> {
        Self::seed(mnemonic, Some(passphrase))
    }
}
```
Now in the main function we can instantiate our seed generator to generate and recover our seed
```rust
fn main() {
    // Instantiate a mnemonic generator with the location of the wordlist
    let mut generator = Bip39Generator::new("english.txt");
    // Generate the mnemonic
    let mnemonic = generator.mnemonic::<16>().unwrap();

    println!("Your mnemonic is: {}", &mnemonic);

    // Recover a wallet by running the mnemonic through seed generator without a passphrase
    let insecure_seed = Bip39Generator::insecure_seed(&mnemonic);

    let passphrase = "BitCoin_iZ_Awesome";
    // Recover a wallet by running the mnemonic through seed generator without a passphrase
    let secure_seed = Bip39Generator::secure_seed(&mnemonic, passphrase);

    // test that the seed generation succeeds
    assert!(&secure_seed.is_ok(),);
    assert!(&insecure_seed.is_ok(),);
}
```
That’s it. We have made life easier for humans :) Cheers!