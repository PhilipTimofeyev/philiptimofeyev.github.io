---
title: Encoding Messages In PNG Files Using Rust
author: philip
date: 2025-04-27 19:16:00 +0800
categories: [Blogging, Tutorial]
tags: [Rust, PNG, Errors, Chunks, JPEG, Cursor, Encoding, Decoding]
render_with_liquid: false
---



The ubiquitous PNG file type makes up a large portion of images thanks to its lossless compression and support for transparency. Lossless compression means we're free to edit the file any number of times and it retains its original quality without becoming blurred and distorted. On the other hand, the PNG's adversary, JPEG, experiences degradation due to downsampling and quantization during the compression process. As an interesting aside, [this study](https://farid.berkeley.edu/downloads/publications/wifs17.pdf) on photo forensics delves deeply into how JPEG compression can reveal photo manipulation.

The second advantage, transparency, is achieved through PNGs utilizing an *alpha* channel along with the typical RGB channels. The alpha channel indicates how transparent each RGB pixel needs to be (an upgrade from the GIF format, which offers transparency as an on/off binary value). Now that we know the main advantages of PNG files, how do they work?

## PNG File Structure

A PNG file is composed of two main components: a header, and a series of chunks.

#### Header

The header is the PNG signature at the beginning of every PNG file, and consists of eight bytes: 

As a decimal: `137 80 78 71 13 10 26 10`

As ASCII: `\211 P N G \r \n \032 \n`

This signature identifies the file as a PNG and helps catch errors during file transfers.

#### Chunks

After the header follows a series of chunks. A chunk is a comprised of four fields, in the following order:

1. **Length** - The number of bytes in the data field represented as a 4-byte unsigned integer. 
2. **Chunk Type** - A 4-byte code, restricted to only uppercase and lowercase ASCII letters.
3. **Chunk data** - The actual data associated with the chunk, can be a length of `0`.
4. **CRC**- A checksum based on the chunk type and chunk data used to validate the chunk and detect errors.



#### A Rudimentary PNG

To help break down what a valid PNG is, we can look at it in its most primative form. The minumum it needs is the header, and three chunks:

1. **Header** - 8-byte signature
2. **IHDR** - The first chunk of every PNG containing the data of the image (height, width, pixel depth, compression,  etc.)
3. **IDAT** - The image's compressed pixel data. 
4. **IEND** - Indicates that there are no more chunks in the image. Contains no data.

A file containing these four parts is the minimum needed to create a valid PNG file.



## Encoding & Decoding Messages

Before we can add any messages to a PNG, we need to construct it using its conventional header and chunk structures from a sequence of bytes.

#### Converting bytes to a PNG struct

The structs we can use to build out the PNG:

```rust
struct Png {
    signature: [u8; 8],
    chunks: Vec<Chunk>,
}

struct Chunk {
    data_length: u32,
    chunk_type: ChunkType,
    chunk_data: Vec<u8>,
    crc: u32,
}

struct ChunkType {
    chunk: [u8; 4],
}
```



Since the PNG file comes in as a series of integer bytes, we need to break down the sequence of bytes into a structured interpretation of a header and chunks. We know the header is the first 8 bytes, we can simply grab them as a vector slice, and then use the `try_into` function to convert the vector into an array of u8s. 

```rust
fn try_from(bytes: &[u8]) -> Result<Self> {
	let header: [u8; 8] = bytes[0..=7].try_into()?;
  //...
}
```

Now we build the chunks:

```rust
fn try_from(bytes: &[u8]) -> Result<Self> {
  //...
  let chunks = &bytes[8..];
  let mut cursor = Cursor::new(chunks);
  let mut all_chunks = Vec::new();

  loop {
      let mut buffer = vec![0; 4];

      if let Err(_e) = cursor.read_exact(&mut buffer) {
          break;
      }
      let chunk_length = u32::from_be_bytes(<[u8; 4]>::try_from(buffer.clone()).unwrap());

      cursor.seek_relative(-4)?;
      buffer = vec![0; (chunk_length + 12) as usize];

      if let Err(_e) = cursor.read_exact(&mut buffer) {
          break;
      }

      let chunk = Chunk::try_from(buffer.as_ref())?;
      all_chunks.push(chunk);
  }
}
```

Here I create three variables:

1. `chunks` - The rest of the bytes of the PNG file, without the header. 
2. `cursor` - Rust's `Cursor` struct to wrap an in-memory buffer and allows it to act like an iterator but with additional features such as seeking back and forth. 
3. `all_chunks` - A vector where we will be appending our chunks as we build them.

To extract a chunk from a sequence of bytes, we need the length of the chunk.

We know that the first part of a chunk is the length of the data as 4 bytes. Within a loop, I set a mutable buffer to a vector of size `4`. When calling the `read_exact(&mut buffer)` function, it reads exactly the amount of bytes that is designated in the buffer vector and sets the cursor to that location in the vector. 

To break out of the loop when the cursor is out of bytes to read, I use `let Err(_e)` to unwrap the `Result` that `read_exact` returns, and if there's an error, which there would be if there are no more bytes to read, it breaks from the loop.

Now that we have our chunk length as a a vector of 4 bytes, we just need to convert it to a `u32` using the `u32::from_be_bytes()` method. This will convert a vector like `[0, 0, 0, 24]` to the u32 `24`.

This gives us the piece of the puzzle we need to extract a single chunk from the sequence of bytes.

Using the cursor's `seek_relative()` function, I move the cursor back to the beginning of the chunk so we can grab it in full. To get the full size, we take our newly acquired `chunk_length` and add it to `12` (4-bytes of length + 4 bytes of chunk type + 4 bytes of CRC), an convert it to a `usize`.

Now we once again use the `read_exact()` function and pass in the length of our chunk, moving the cursor to the start of the next chunk while grabbing all of the bytes that make up a valid chunk and pushing them to the `buffer`. 

Once we have a chunk in byte form, we use a similar process to build a `chunk` struct. Since we know the first 4 bytes are the length, the next 4 bytes are the chunk type, and the last 4 bytes are the CRC, we have everything we need to construct the struct.

We continue looping this process, building each chunk and pushing it to a vector, until the end of the file. 

Finally we've built a PNG struct from a stream of bytes!

```rust
let png = Png {
    signature: Png::STANDARD_HEADER,
    chunks: all_chunks,
};
```



#### Adding a Message

To add a message, we need to pass a valid chunk type as the "key", and a message as a string. Let's create a key using a chunk type.

##### Creating a Key (ChunkType)

```rust
impl FromStr for ChunkType {
    type Err = Error;

    fn from_str(s: &str) -> Result<Self> {
        let chunk_arr = match s.as_bytes().try_into() {
            Ok(result) => result,
            Err(_) => {
              let error_message = "Chunk type (key) must be a length of 4 characters.".to_string();
            	return Err(ChunkTypeError::InvalidLength(error_message).into())
            }
        };

        let chunk_type = ChunkType { chunk_type: chunk_arr };
        chunk_type.is_valid()
    }
}
```

When we pass a string key to the `ChunkType::from_str()` , we first convert the string into an array of four u8 bytes. Because `as_bytes()` returns a `&[u8]`, we need to convert it to a `[u8; 4]`  using the `try_into()` method, which returns an error if the length of the key is not `4`.

Then we create a `ChunkType` object and check if it's valid. The `is_valid()` method (not defined here) checks if the key contains only ASCII alphabet characters and that the third character is uppercase.

##### Creating a Chunk

```rust
pub fn new(chunk_type: ChunkType, data: Vec<u8>) -> Chunk {
    let data_length = data.len();
    let crc = get_crc(&chunk_type, &data);

    Chunk {
        data_length: data_length.try_into().unwrap(),
        chunk_type,
        chunk_data: data,
        crc,
    }
}
```

To create a chunk we pass in our newly created chunk type, as well as a the message as vector of u8 bytes converted from a `String` using the `as_bytes()` method. 

These two pieces are what we need to create a chunk because the `length` is created using the length of the `data` (message), and the `crc` is created on our end (not shown here). 

Now we have a valid chunk with our key and secret message!

##### Adding Message to the File

The final part of our encoding is to add our valid chunk to the PNG structure, and then re-write the file.

```rust
pub fn encode(args: EncodeArgs) -> Result<()> {
    let png_file = read(&args.filepath)?;
    let mut png_struct = Png::try_from(&png_file[..])?;

    let chunk_type = ChunkType::from_str(&args.chunk_type)?;
    let chunk = Chunk::new(chunk_type, args.message.as_bytes().into());

    png_struct.append_chunk(chunk);

    let current_dir = env::current_dir()?;

    match args.output_file {
        Some(output_path) => {
            let file_path = current_dir.join(output_path);
            fs::write(file_path, png_struct.as_bytes())?;
        }
        None => fs::write(args.filepath, png_struct.as_bytes())?,
    }

    Ok(())
}

```

After creating a PNG structure from the file, we use the `append_chunk()` method to add our chunk to the end of the sequence of chunks, right before the last chunk `IEND` to retain the conventional order.

Now we just write the modified PNG object either as a new file, or modify the original PNG file.

A PNG file with our encoded message is now created! But how can we decode it?

#### Decoding a Message

Decoding a PNG file is a similar process to encoding, except we just need the original key (chunk type) to reveal the message.

```rust
pub fn decode(args: DecodeArgs) -> Result<()> {
    let png_file = read(&args.filepath)?;
    let png_struct = Png::try_from(&png_file[..])?;
    let decoded_message =
        png_struct
            .chunk_by_type(args.chunk_type.as_str())
            .ok_or(chunk::ChunkError::NotFound(
                "Chunktype not found.".to_string(),
            ))?;

    println!("{}", decoded_message.data_as_string()?);

    Ok(())
}

/// png.rs module
    pub fn chunk_by_type(&self, chunk_type: &str) -> Option<&Chunk> {
        self.chunks()
            .iter()
            .find(|chunk| chunk.chunk_type.to_string() == chunk_type)
    }
```

After creating a PNG object using our PNG struct, we call the `chunk_by_type()` method, passing in the chunk type as a string slice, and then using the `find()` method to search through the chunks and find one with that specific chunk type. Then returning an `Option<T>` which I convert to a `Result<T, E>` and return either the first chunk associated with the chunk type, or a custom `ChunkError`. 

Finally, the decoded message is printed using the `data_as_string()` method which converts the vector of `u8` bytes to a `String` which we can read!

#### Closing Thoughts

Many file formats utilize chunk structure so hopefully this helped break down and explain some of the PNG and chunk structure concepts through the language of Rust. 



Sources:

- [pngme_book](https://jrdngr.github.io/pngme_book/)
